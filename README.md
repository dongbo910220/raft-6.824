
## LAB 1

首先，理解整个过程, 通过文件分成M个部分，对每个部分通过doMap()处理成N个中间文件，接下来N个reduce程序每次处理M个(下图有误)中间文件（使用doReduce），（比如统计单词个数），形成N个等待merge文件，最后merge形成最终的文件完成处理

![preview](https://pic1.zhimg.com/v2-ace321106e99a46e5aa8738108733478_r.jpg)

#### part 1

实现doMap ，方法是把文件InFile读进来，随后生成一个二维数组存放根据HASH分出来的各个词的位置（在内存里），按照约定生成一个个临时文件，利用JSON格式转化，随后写入磁盘里就好了

```go
func doMap(
	jobName string, // the name of the MapReduce job
	mapTask int, // which map task this is
	inFile string,
	nReduce int, // the number of reduce task that will be run ("R" in the paper)
	mapF func(filename string, contents string) []KeyValue,
) {
	content, err:= ioutil.ReadFile(inFile)
	if err != nil {
		panic(err)
	}
	keyValues := mapF(inFile, string(content))
	allKeyValues := make([][]KeyValue, nReduce)
	for _, pair := range keyValues{
		key, value := pair.Key, pair.Value
		n := ihash(key) % nReduce
		allKeyValues[n] = append(allKeyValues[n], KeyValue{key, value})
	}

	for i := 0; i < nReduce; i++{
		intermediateFileName := reduceName(jobName, mapTask, i)
		file, err := os.Create(intermediateFileName)
		if err != nil {
			panic(err)
		}
		taskJson, err := json.Marshal(allKeyValues[i])
		if err != nil {
			panic(err)
		}
		if _, err := file.Write(taskJson); err != nil {
			panic(err)
		}
		if err := file.Close();  err != nil {
			panic(err)
		}
	}
}
```

doReduce  先把所负责的M个文件统一放到一起（allKeyValuePairs），把相同key的pair的value加到同一个队列里，随后调用reduceF 对队列进行处理计算结果，再保存生成文件即可。

```go
func doReduce(
	jobName string, // the name of the whole MapReduce job
	reduceTask int, // which reduce task this is
	outFile string, // write the output here
	nMap int, // the number of map tasks that were run ("M" in the paper)
	reduceF func(key string, values []string) string,
) {
	allKeyValuePairs := make([]KeyValue, 0)
	for mapTask := 0; mapTask < nMap; mapTask++ {
		currentTaskFileName := reduceName(jobName, mapTask, reduceTask)
		currentTask, err := ioutil.ReadFile(currentTaskFileName)
		if err != nil {
			panic(err)
		}
		currentFileKeyPairs := make([]KeyValue, 0)
		if err := json.Unmarshal(currentTask, &currentFileKeyPairs); err != nil{
			panic(err)
		}
		for _, pair := range currentFileKeyPairs {
			allKeyValuePairs = append(allKeyValuePairs, pair)
		}
	}

	//do not need sort. map is better
	//sort.Sort(KeyValuePairs(allKeyValuePairs))
	KeyValuesmap := make(map[string][]string)
	for _, pair := range(allKeyValuePairs) {
		key, value := pair.Key, pair.Value
		KeyValuesmap[key] = append(KeyValuesmap[key], value)
	}

	outputFile, err := os.Create(outFile)
	if err != nil {
		panic(err)
	}
	enc := json.NewEncoder(outputFile)
	for key, values := range KeyValuesmap {
		if err := enc.Encode(KeyValue{key, reduceF(key, values)}); err != nil {
			panic(err)
		}
	}
	if err := outputFile.Close(); err != nil {
		panic(err)
	}
}
```

#### part 2

 重写 mapF 把content 转化成一系列的keyvalue,  reduceF 把[] string 计算总和返回即可（中间转换下格式）

```go
func mapF(filename string, contents string) []mapreduce.KeyValue {
	// Your code here (Part II).
	f := func(c rune) bool {
		return !unicode.IsLetter(c) && !unicode.IsNumber(c)
	}
	strs:=strings.FieldsFunc(contents, f)
	res:=[]mapreduce.KeyValue{}
	for _,str :=range strs{
		res = append(res, mapreduce.KeyValue{str,"1"})
	}
	return res
}
func reduceF(key string, values []string) string {
	// Your code here (Part II).
	res:= 0
	for _,str:=range values{
		i, err := strconv.Atoi(str)
		if err!=nil{
			panic(err)
		}
		res+=i
	}
	return strconv.Itoa(res)
}
```

#### part 3 Distributing MapReduce tasks

多开几个go程去发活即可，随后调用rpc等待返回

```go
func schedule(jobName string, mapFiles []string, nReduce int, phase jobPhase, registerChan chan string) {
	var ntasks int
	var n_other int // number of inputs (for reduce) or outputs (for map)
	switch phase {
	case mapPhase:
		ntasks = len(mapFiles)
		n_other = nReduce
	case reducePhase:
		ntasks = nReduce
		n_other = len(mapFiles)
	}

	fmt.Printf("Schedule: %v %v tasks (%d I/Os)\n", ntasks, phase, n_other)
	// Your code here (Part III, Part IV).
	wg := sync.WaitGroup{}
	wg.Add(ntasks)
	i := 0
	for {
		if i >= ntasks {
			break
		}
		worker := <- registerChan
		go func(worker string, i int ) {
			args := DoTaskArgs{}
				fmt.Printf("the current worker (%v) is dealing with %v  %v\n", worker , phase, i)
			switch phase {
			case mapPhase:
				args = DoTaskArgs{
					JobName: jobName,
					File:	 mapFiles[i],
					Phase:   mapPhase,
					TaskNumber:	i,
					NumOtherPhase:	n_other,
				}
			case reducePhase:
				args = DoTaskArgs{
					JobName: jobName,
					File:	 "",
					Phase:   reducePhase,
					TaskNumber:	i,
					NumOtherPhase:	n_other,
				}
			}

			if ok := call(worker, "Worker.DoTask",args, nil); !ok{
				fmt.Print("impossible")
			}
			wg.Done()
			registerChan <- worker
		}(worker, i)
		i += 1
	}
	wg.Wait()
	fmt.Printf("Schedule: %v done\n", phase)
}
```

#### part 4 处理worker failure

当 worker failure时，反复再发即可

```go
ok := call(worker, "Worker.DoTask",args, nil)
			for ok == false {
				worker = <- registerChan
				fmt.Printf("the current worker (%v) is dealing with %v  %v\n", worker , phase, i)
				ok = call(worker, "Worker.DoTask",args, nil)
			}
```

#### part 5 实现倒排索引

mapF 中直接存得不是“1” 计数器了，而是每个单词对应的document,注意去重就好。

reduceF中 统计出现的document总数，再转化指定格式就好~

```go
func mapF(document string, value string) (res []mapreduce.KeyValue) {
	// Your code here (Part V).
	f := func(c rune) bool {
		return !unicode.IsLetter(c)
	}
	strs := strings.FieldsFunc(value, f)
	res = []mapreduce.KeyValue{}
	maptmp := make(map[string]bool)
	for _, str := range strs {
		maptmp[str] = true
	}

	for str, _ := range maptmp{
		res = append(res, mapreduce.KeyValue{str, document})
	}
	return
}

func reduceF(key string, values []string) string {
	// Your code here (Part V).
	s := strconv.Itoa(len(values)) + " " + strings.Join(values, ",")
	return s
}
```



## 论文提炼

#### 5.1 raft 基础

![image-20211002115311755](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002115311755.png)

有关身份转换的图见上面，每一个term里面会分成两部分，一部分是leader选举阶段。另一部分则是正常操作的阶段（如果选举没成功则没有这个阶段了）。每一个服务器节点存储一个当前任期号，该编号随着时间单调递增。服务器之间通信的时候会交换当前任期号；如果一个服务器的当前任期号比其他的小，该服务器会将自己的任期号更新为较大的那个值。如果一个 candidate 或者 leader 发现自己的任期号过期了，它会立即回到 follower  状态。如果一个节点接收到一个包含过期的任期号的请求，它会直接拒绝这个请求。

#### 5.2 leader选举

+ Each server will vote for at most one candidate in a given term, on a first-come-first-served basis （每个任期一个服务器只能投一次）

+ 成为leader 后就可以给其他server 发送心跳来声明自己已经是leader, 避免开启下一轮的选举

+ 在一个candidate等待投票的时候，如果收到了 appendentries rpc from 其他server 声明自己已经成为leader， 接下来判断该 appendentries 的 term , 如果小于自己的，则拒绝，接着保留自己的candidate身份，>= 自己的term 则转变为 follower

+ 如果投票分开了，以至于没有人能获得大多数，那么每个candidate就会开启一个新的选举（by incrementing its term）， 并发起新一轮的 RequestVote RPCs （但这个过程可能造成无限重复）

  + 为了解决可能限重复这个问题，就设计randomized election timeouts 随机选举时间来进行规避，这样就可以在时间到期的时候（而另一些candidate没到）直接开启新一轮的领导选举（with incremented term）来增大获胜的可能性，从而避免这个问题

总结：本质上这块的设计思路是一个等级系统，高等级的更容易用，但也带来了个新的问题（(a lower-ranked server might need to time out and become a candidate again if a higher-ranked server fails, but if it does so too soon, it can reset progress towards electing a leader)）组后改用随机重试加以解决 可能竞争太多次的问题

log entry

每一个log 都包含一个当前leader 所在的term, 当然也有一个自己在log中的序号。

leader 最终会决定一个log 的执行， （.Raft guarantees that committed entries are durable

and will eventually be executed by all of the available state machines）raft 算法会保证 执行了的 log 最终会持久化并被每一个可用的状态机执行。执行的时机是一个log 被大多数的状态机复制了（同时之前的也会提交）， 一旦一个follower 得知了一个Log被commited了 ，它也会apply it.

raft 协议保证：如果不同日志中的两个条目拥有相同的索引（index）和任期号(term)，那么他们之前的所有日志条目（包括本条）也都相同。在发送 AppendEntries RPC 的时候，leader 会将前一个日志条目的索引位置和任期号包含在里面。如果 follower 在它的日志中找不到包含相同索引位置和任期号的条目，那么他就会拒绝该新的日志条目，空的时候肯定满足这个 Log matching property, 就像数学里的归纳一样。

如果出现日志冲突的情况，那么leader 会让follower 重写有关日志（具体有关规定见下节）

The leader maintains a *nextIndex* for each follower, which is the index of the next log entry the leader will send to that follower. 同样，为了处理掉当leader log 与 follower log 不一致的情况，每个leader 都拥有一个nextIndex 给各个 follower 准备好， 当一个candidate刚刚成为Leader 的时候，他会给每个Follower 统一初始化成 自己的下一个。 如果follower 的不一致，那么他就会reject 掉， 同时leader 会把当前的follower 的 nextIndex  -1 再次发送， 直到成功（这是会删除掉之前冲突的）（如果嫌一个个检查太慢 ，可以一次发当前任期的第一个，然后依次递减任期，但事实上这个作用相当有限，因为不一致不常发生）

#### safety

假设一个follower 变成了不可用的状态，而leader 提交了几条log , 之后这个follower 又变成了可用并且成功变成了leader， 之后leader 就接着提交了一些entry, 这就导致了状态机的状态不一样。所以就要新加一条限制，让这个follower 选不上即可。（ The restriction ensures that the leader for any given term contains all of the entries committed in previous terms (the Leader Completeness Property from Figure 3). 也就是任何一个当选的leader 都必须包含之前term 所有提交了的entries （但此时该Leader 不一定提交，个人所见），所以这就导致了一个好处，就是log entries 只会从 leader 流向 follower 而不会相反。

那么从 candidate 选举成 leader 的地方开始做文章比较好， 一个candidate 要想成为一个leader 就需要得到大多数的支持，而之前提交的entry 一定也被大多数所“拥有”， 这就导致了一个candidate 必须具有至少和大多数一样的 最新的 entry，这就保证了他会拥有所有的 commited entry （但某些情况他可能还未提交） （The RequestVote RPC implements this restriction: the RPC includes information about the candidate’s log, and the voter denies its vote if its own log is more up-to-date than that of the candidate.） 就是candidate 要投票的时候把自己的最近的log 带上即可， 如果voter 发现还不如自己的新呢，则拒绝投票。那么，这里的最新是啥意思呢，就是比当前Log 中最近的那一条，如果terms 不同，那么terms 大的更“新”，如果相同，那么就比Index.

接下来讲讨论一种较为罕见的情况， 如果一个leader 还没来的及提交entry 的时候就先挂掉了，那么这是大多数已经复制了该条Log, 新上任的leader 就会提交此条Log , 但是新leader 如何得知上一条log 提交与否呢？ 但是这时不一定能提交，没准还是会出现一些新情况，详见图8（*待补充*）

为了解决图8的情况（由 c -> d 中， 如果判断副本数目，那么确实2已经提交了， 但是事实上没有），raft 提交之前任期的Log 不会采取计算副本数目的方式，只有当前任期才会计算复本数目。一旦一个当前任期的提交了，那么根据之前介绍的原则，之前的entry 也就因此间接提交了。 raft 解决这个的办法就是在log entry 中增加对应的term 号。 （这有啥作用）

如果 follower 掉线了怎么办，很简单，接着发，因为消息满足幂等性， 当发现自己已经接收过的时候就无视就好了

#### 时间信息

Leader 选举是 Raft 中定时最为关键的方面。 只要整个系统满足下面的时间要求，Raft 就可以选举出并维持一个稳定的 leader：

> 广播时间（broadcastTime） << 选举超时时间（electionTimeout） << 平均故障间隔时间（MTBF）

#### 日志压缩

长时间的相应client 的请求会让日志越来愈多，超过限制，那么那些已经提交的日志已经变成了状态，干脆直接把状态存下来，省下很多磁盘空间，但是如果有新的 follower 加入怎么办呢，一个显而易见的方式就是把 快照直接传给他，再把快照之后的日志也传给他就好 了 -- 以上为看文章实现之前的种种猜测

为了把快照和 log 对应起来， 需要记录是写到哪的快照（term 和 index）

我还猜了半天咋实现呢， 原来是新设计了一个RPC （The leader uses a new RPC called InstallSnapshot to send snapshots to followers that are too far behind;）

快照的方式违反了 Raft 的 strong leader 原则，因为 follower 可以在不知道 leader  状态的情况下创建快照。但是我们认为这种违背是合乎情理的。Leader  的存在，是为了防止在达成一致性的时候的冲突，但是在创建快照的时候（发送快照的前提是已经得知 follower 已经能靠发 log 来解决了），一致性已经达成，因此没有决策会冲突。 如果采用只有 leader 能创建快照的方式，那么发送快照还是很占据网络带宽的（因为follower 也会出现log 太多，太占据空间了，所以也需要快照，还是本地创建更容易），不太划算。另外leader的实现更为复杂。

那么何时创建快照呢，采用当日志大小达到一个固定大小的时候开始创建。但创建的同时会影响正常的操作，不希望它影响正常的操作，所以采用写时复制即使。这样新的更新就可以在不影响正在写的快照的情况下被接收了（linux 可以fork 一下来解决， 父进程继续，而子进程开始写快照就行了）

#### 客户端交互

client 给任意一个server 发送请求，如果不是 Leader 那么就会把自己目前得知的leader 交给他（AppendEntries 包含了leader 作为发送者的 ip 地址），如果leader 崩溃了， 那么就会超时，客户端会再次挑选服务器重试。

如果一个目录提交后再返回给client 之前突然挂了，那么client 会再次发送， 这就导致了会执行同一个命令多次的情况，给一个命令唯一的命令一个 seq no, 这样就可以避免（直接返回结果， 不再执行）



#### 2A 实现领导人选举和心跳

实现领导人选举和心跳。第2A部分的目标是选出一个leader，如果没有失败，则leader继续担任leader，如果旧leader失败，或者发送给/接收旧leader的包丢失，则由新leader接管。 

修改Make()以创建一个后台goroutine，当它有一段时间没有收到其他peer的消息时，通过发送RequestVote rpc来定期启动领导人选举。这样同伴就会知道谁是领导者，如果已经有了领导者，或者成为领导者本身。实现RequestVote() RPC处理程序，以便服务器可以为彼此投票。

写一个LogEntry, 包含一个操作 以及当前leader 所在的任期（ ==待补充== ）

```go
type LogEntry struct {
    Command interface{}
    Term    int
}
```

定义心跳时间以及 peer 的status

```go
const (
	Follower = 1
	Candidate = 2
	Leader = 3
	HEART_BEAT_TIMEOUT = 100 //心跳超时，要求1秒10次，所以是100ms一次
)
```

定义 logEntry

```go
type LogEntry struct {
	Command interface{}
	Term    int
}
```

实现raft 结构, 几乎就是把figure 2 的图照着写一遍就行

```go
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]

	// Your data here (2A, 2B, 2C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.
	timer *time.Timer 		// 定时器
	timeout time.Duration	// 选举超时周期 election timeout
	state	int 			//角色

	appendCh chan bool   	// 心跳
	voteCh	 chan bool 
	voteCount int

	//Persistent state on all servers: (Updated on stable storage before responding to RPCs)
	//from figure 2
	currentTerm	int 		//latest term server has seen (initialized to 0 on first boot, increases monotonically)
	votedFor    int        //candidateId that received vote in current term (or null if none)
    log         []LogEntry //log entries; each entry contains command for state machine, and term when entry was received by leader (first index is 1)

	//Volatile state on all servers:
    commitIndex int //index of highest log entry known to be committed (initialized to 0, increases monotonically)
    lastApplied int //index of highest log entry applied to state machine (initialized to 0, increases monotonically)
    //Volatile state on leaders:(Reinitialized after election)
    nextIndex  []int //for each server, index of the next log entry to send to that server (initialized to leader last log index + 1)
    matchIndex []int //for each server, index of highest log entry known to be replicated on server (initialized to 0, increases monotonically)

}
```

![image-20211002111842887](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002111842887.png)

实现 getstate(), 多线程 涉及到公用变量的读取等都加锁

```go
func (rf *Raft) GetState() (int, bool) {

	var term int
	var isleader bool
	// Your code here (2A).
	rf.mu.Lock()
	defer rf.mu.Unlock()
	term = rf.currentTerm
	isleader = rf.state == Leader
	return term, isleader
}
```

实现 RequestVoteArgs 以及 RequestVoteReply

```go
type RequestVoteArgs struct {
	// Your data here (2A, 2B).
	Term         int //candidate’s term
    CandidateId  int //candidate requesting vote
    LastLogIndex int //index of candidate’s last log entry (§5.4)
    LastLogTerm  int //term of candidate’s last log entry (§5.4)
}

type RequestVoteReply struct {
	// Your data here (2A).
	Term        int  //currentTerm, for candidate to update itself
    VoteGranted bool //true means candidate received vote
}
```

![image-20211002112028705](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002112028705.png)

实现 RequestVote, 主要注意上图得到 receiver implementation,  如果大于自己的term 则直接转成follower，其中第二条就是candidate 要投票的时候把自己的最近的log 带上即可， 如果voter 发现还不如自己的新呢，则拒绝投票。那么，这里的最新是啥意思呢，就是比当前Log 中最近的那一条，如果terms 不同，那么terms 大的更“新”，如果相同，那么就比Index. Make() 里 candidate不处理这种<-voteCh 这种情况, 只有follower处理

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()
	DPrintf("Candidate[raft%v][term:%v] request vote: raft%v[%v] 's term%v\n", args.CandidateId, args.Term, rf.me, rf.state, rf.currentTerm)
	reply.VoteGranted = false 
	//Reply false if term < currentTerm (5.1)
	if args.Term < rf.currentTerm {
		DPrintf("raft%v don't vote for raft%v\n", rf.me, args.CandidateId)
		reply.Term = rf.currentTerm
		reply.VoteGranted = false 
		return 
	}
	//If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.switchStateTo(Follower)
	}
	// 有个疑问 要是不满足要求不直接返回么， 也算收到么，还是无所谓，这块不明确 还需推敲, 再看看论文 经过看论文 自行裁决 不处理就得了
	//If votedFor is null or candidateId, and candidate’s log is at least as up-to-date as receiver’s log, grant vote
	if rf.votedFor == -1 || rf.votedFor == args.CandidateId { // 待补充  (rf.log[rf.lastApplied].term <= args.lastLogTerm 如果相等继续判断 rf.lastApplied <= args.lastLogIndex 否则就直接进)
		DPrintf("raft%v vote for raft%v\n", rf.me, args.CandidateId)
        reply.VoteGranted = true
        rf.votedFor = args.CandidateId
	}
	reply.Term = rf.currentTerm
	go func(){
		rf.voteCh <- true                // 阻塞在这里 所以要开一个新线程
	}()
}
```

发送方式， rpc 远程调用, 成功返回true 否则就是false (超时等等情况都算)

```go
func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	return ok
}
```

AppendEntriesArgs 发送心跳以及logs 

![image-20211002121044222](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002121044222.png)

```go
//Invoked by leader to replicate log entries (§5.3); also used as heartbeat (§5.2).
type AppendEntriesArgs struct {
    Term         int        //leader’s term
    LeaderId     int        //so follower can redirect clients
    PrevLogIndex int        //index of log entry immediately preceding new ones 在发送 AppendEntries RPC 的时候，leader 会将前一个日志条目的索引位置和任期号包含在里面。如果 follower 在它的日志中找不到包含相同索引位置和任期号的条目，那么他就会拒绝该新的日志条目，空的时候肯定满足这个 Log matching property
    PrevLogTerm  int        //term of prevLogIndex entry
    Entries      []LogEntry //log entries to store (empty for heartbeat; may send more than one for efficiency)
    LeaderCommit int        //leader’s commitIndex
}
type AppendEntriesReply struct {
    Term    int  //currentTerm, for leader to update itself
    Success bool //true if follower contained entry matching prevLogIndex and prevLogTerm
}
```

以及func AppendEntries, 目前仅仅满足第一条就够了（不涉及log）如果之前我是一个candidate,如果是appendCh 是leader在 同一个term发的（说明它刚刚竞选成功）， 那么要在Make里进行身份转换成Follower

![image-20211002121301879](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002121301879.png)

```go
func (rf *Raft) AppendEntries (args *AppendEntriesArgs, reply *AppendEntriesReply){
	rf.mu.Lock()
	defer rf.mu.Unlock()
	DPrintf("leader[raft%v][term:%v] beat term:%v [raft%v][%v]\n", args.LeaderId, args.Term, rf.currentTerm, rf.me, rf.state)
	reply.Success = true 
	if args.Term < rf.currentTerm {
		//1. Reply false if term < currentTerm (§5.1)
		reply.Success = false
        reply.Term = rf.currentTerm
        return                 // 不符合要求 直接返回 不会 rf.appendCh <- true
	} else if args.Term > rf.currentTerm {  
        //If RPC request or response contains term T > currentTerm:set currentTerm = T, convert to follower (§5.1)
        rf.currentTerm = args.Term
        rf.switchStateTo(Follower)
    }
	go func() {
		rf.appendCh <- true  // 表示收到 appendentry 了, 而且是符合要求的（任期大于当前任期）， 会再开一个goroutine 来处理
	}()
}
```

func sendAppendEntries 与之前类似， 略

接下来就是实现Make()了, 一开始所peer 的状态都是Follower, 设置开始的term, votedfor 等等，完成chan 的初始化， 设置超时时间（不能采取文章中的 150 ~300，变成 300~ 400）以及timer,  随后开一个新的goroutine （因为有要求make 需要立即返回），来循环监听各个事件（根据身份的变化），每次都重新设置随机超时时间（300 ~ 400）。

+ 如果是follower 那么接到任意的RPC都算接到通信，重置定时器即可，否则如果定时器到时间以后就会自动变成Candidate, 并立刻发起投票。
+ 若是Candidate，则不管voteCh的内容（在RPC所对应的远程调用程序中，一旦发现自己term < args.term的时候自动转换成Follower, 不需要操心），如果到时间， 就默认发起新一轮的投票（currentTerm + 1），默认选项是属票数，一旦到了就变成leader， 这里可以考虑睡下(比如10ms)，要不太占用cpu
+ 若是leader 则不管两个通道的内容(Votech & appendCh)（在RPC所对应的远程调用程序中，一旦发现自己term < args.term的时候自动转换成Follower, 不需要操心），只管heartbeats,这里没设计好，应该是在这里直接睡比较好（100ms），不应该放在heartbeats里

```go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	// Your initialization code here (2A, 2B, 2C).
	DPrintf("create raft%v...", me)
	rf.state = Follower
	rf.currentTerm = 0
	rf.votedFor = -1
	rf.voteCh = make(chan bool)
	rf.appendCh = make(chan bool)

	electionTimeout := HEART_BEAT_TIMEOUT * 3 + rand.Intn(HEART_BEAT_TIMEOUT)
	rf.timeout = time.Duration(electionTimeout) * time.Millisecond
	DPrintf("raft%v's election timeout is:%v\n", rf.me, rf.timeout)
	rf.timer = time.NewTimer(rf.timeout)
	go func ()  {
		for {
			rf.mu.Lock()
			state := rf.state
			rf.mu.Unlock()
			electionTimeout := HEART_BEAT_TIMEOUT*3 + rand.Intn(HEART_BEAT_TIMEOUT)
            rf.timeout = time.Duration(electionTimeout) * time.Millisecond

			switch state {
			case Follower:
				//如果在选举超时之前都没有收到领导人的心跳，或者是候选人的投票请求，就自己变成候选人
				select {
				case <- rf.appendCh:
					rf.time.Reset(rf.timeout)
				case <- rf.voteCh:
					rf.time.Reset(rf.timeout)
				case <- rf.timer.C:
					rf.mu.Lock()
					rf.switchStateTo(Candidate)
					rf.mu.Unlock()
					rf.startElection()
				}
			case Candidate:
				select {
				case <- rf.appendCh:
					rf.timer.Reset(rf.timeout)
					rf.mu.Lock()
                    rf.switchStateTo(Follower)
                    rf.mu.Unlock()
				case <-rf.timer.C:
                    rf.startElection()
				default:
                    rf.mu.Lock()
                    if rf.voteCount > len(rf.peers)/2 {
                        rf.switchStateTo(Leader)
                    }
                    rf.mu.Unlock()
                }
			case Leader:
                //heartbeats
                rf.heartbeats()
			}
		}	
	}()
	// initialize from state persisted before a crash
	rf.readPersist(persister.ReadRaftState())

	return rf
}
```

leader 发heartbeats, 为了不发一个等一个（call就要等） 直接启goroutine 发送 同使如果接到反馈 reply.Term > rf.currentTerm 就自动转为Follower, 发完睡一下time.Sleep..

```go
func (rf *Raft) heartbeats() {
	for peer, _ := range rf.peers {
		if peer != rf.me {
			go func(peer int) {
				args := AppendEntriesArgs{}
				rf.mu.Lock()
				args.Term = rf.currentTerm
				args.LeaderId = rf.me
				args.Term = rf.currentTerm
				args.LeaderId = rf.me
				rf.mu.Unlock()
				reply := AppendEntriesReply{}
				if rf.sendAppendEntries(peer, &args, &reply) {
					//If RPC request or response contains term T > currentTerm:set currentTerm = T, convert to follower (§5.1)
					rf.mu.Lock()
					if reply.Term > rf.currentTerm {
						rf.currentTerm = reply.Term
						rf.switchStateTo(Follower)
					}
					rf.mu.Unlock()
				}
			}(peer)
		}
	}
	time.Sleep(HEART_BEAT_TIMEOUT * time.Millisecond)
}
```

switchStateTo, 如果相等直接返回， 否则就进行转换，由调用者加锁，原因是这可能是rf中很多转变中的一步，索性由调用者统一加锁

```go
//切换状态，调用者需要加锁
func (rf *Raft) switchStateTo(state int) {
    if rf.state == state {
        return
    }
    if state == Follower {
        rf.state = Follower
        DPrintf("raft%v become follower in term:%v\n", rf.me, rf.currentTerm)
        rf.votedFor = -1
    }
    if state == Candidate {
        DPrintf("raft%v become candidate in term:%v\n", rf.me, rf.currentTerm)
        rf.state = Candidate
    }
    if state == Leader {
        rf.state = Leader
        DPrintf("raft%v become leader in term:%v\n", rf.me, rf.currentTerm)
    }
}
```

Candidate 的startElection,  每次自动更新当前的term (rf.currentTerm += 1), 然后投自己，重设超时时间，继续开启goroutine 向每一个发送 RequestVote, 如果投自己了就得票数 + 1（rf.voteCount += 1） 同使如果接到反馈 reply.Term > rf.currentTerm 就自动转为Follower， 这里不统计票， 靠主线程来统计

```go
func (rf *Raft) startElection() {
    rf.mu.Lock()
    //    DPrintf("raft%v is starting election\n", rf.me)
    rf.currentTerm += 1
    rf.votedFor = rf.me //vote for me
    rf.timer.Reset(rf.timeout)
    rf.voteCount = 1
    rf.mu.Unlock()
    for peer, _ := range rf.peers {
        if peer != rf.me {
            go func(peer int) {
                rf.mu.Lock()
                args := RequestVoteArgs{}
                args.Term = rf.currentTerm
                args.CandidateId = rf.me
                // DPrintf("raft%v[%v] is sending RequestVote RPC to raft%v\n", rf.me, rf.state, peer)
                rf.mu.Unlock()
                reply := RequestVoteReply{}
                if rf.sendRequestVote(peer, &args, &reply) {
                    rf.mu.Lock()
                    if reply.VoteGranted {
                        rf.voteCount += 1
                    } else if reply.Term > rf.currentTerm {
                        //If RPC request or response contains term T > currentTerm:set currentTerm = T, convert to follower (§5.1)
                        rf.currentTerm = reply.Term
                        rf.switchStateTo(Follower)
                    }
                    rf.mu.Unlock()
                } else {
                    // DPrintf("raft%v[%v] vote:raft%v no reply, currentTerm:%v\n", rf.me, rf.state, peer, rf.currentTerm)
                }
            }(peer)
        }
    }
}
```



**以定时器的角度重写一遍**，好处是直接在Requestvote & AppendEntries RPC 可以直接reset 各自的timer , 避免在主线程重新修改， 另外把选举时间相关的重置代码抽取出来

```go
type Raft struct {
	// appendCh  chan bool // 心跳  注掉
	// voteCh    chan bool          注掉
    electionTimer  *time.Timer // 选举定时器
	heartbeatTimer *time.Timer // 心跳定时器
}
func randTimeDuration() time.Duration {
    return time.Duration(HEART_BEAT_TIMEOUT*3+rand.Intn(HEART_BEAT_TIMEOUT)) * time.Millisecond
}
```

取消掉之前的chan 通信方式即可

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
   /*go func() {
		rf.voteCh <- true // 阻塞在这里 所以要开一个新线程
	}() 不用再采取这种方式进行线程间的交换 直接充值定时器就好了*/
    rf.electionTimer.Reset(randTimeDuration())
}

func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
    /*go func() {
		rf.appendCh <- true // 表示收到 appendentry 了, 而且是符合要求的（任期大于当前任期）， 会再开一个goroutine 来处理
	}() 不用再采取这种方式进行线程间的交换 直接充值定时器就好了 */
	rf.electionTimer.Reset(randTimeDuration())
}
    
```

改变Make() 注掉votech appendch 相关的 重写主线程（监听作用）

+ 当electionTimer 到了的时候

  + 如果是Follower 就转变为candidate ， 在转换函数中 如果转换成candidate会自动发起投票
  + 如果是Candidate 就再次发起投票

+ 当heartbeatTimer 到了的时候
  + 只有leader 会响应 发送心跳并重置 heartbeatTimer

```go
func Make(peers []*labrpc.ClientEnd, me int,
    persister *Persister, applyCh chan ApplyMsg) *Raft {
	// rf.voteCh = make(chan bool)
	// rf.appendCh = make(chan bool)

	// electionTimeout := HEART_BEAT_TIMEOUT*3 + rand.Intn(HEART_BEAT_TIMEOUT)
	// rf.timeout = time.Duration(electionTimeout) * time.Millisecond
	// DPrintf("raft%v's election timeout is:%v\n", rf.me, rf.timeout)
	// rf.timer = time.NewTimer(rf.timeout)
	rf.heartbeatTimer = time.NewTimer(HEART_BEAT_TIMEOUT * time.Millisecond)
    rf.electionTimer = time.NewTimer(randTimeDuration())
    go func() {
		for {
			select {
			case <-rf.electionTimer.C:
				rf.mu.Lock()
				switch rf.state{ 
				case Follower:
					rf.switchStateTo(Candidate)
				case Candidate:
					rf.startElection()
				}
				rf.mu.Unlock()
				
			case <- rf.heartbeatTimer.C:
				rf.mu.Lock()
				if rf.state == Leader {
                    rf.heartbeats()
                    rf.heartbeatTimer.Reset(HEART_BEAT_TIMEOUT * time.Millisecond)
                }
				rf.mu.Unlock()
			}
		}
	}()
}
```

heartbeats 注销掉

```go
//time.Sleep(HEART_BEAT_TIMEOUT * time.Millisecond)
```

重写 switchStateTo

+ 如果是follower , 说明刚刚由其他转为 follower 要把投票记为-1（为投票状态），重置选举时间
+ Candidate 成为候选人后立马进行选举
+ 成为leader后要 停止electionTimer,(其实这里不关也行)

```go
//切换状态，调用者需要加锁
func (rf *Raft) switchStateTo(state int) {
    if state == rf.state {
        return
    }
    DPrintf("Term %d: server %d convert from %v to %v\n", rf.currentTerm, rf.me, rf.state, state)
    rf.state = state
    switch state {
    case Follower:
        rf.heartbeatTimer.Stop()
        rf.electionTimer.Reset(randTimeDuration())
        rf.votedFor = -1
    case Candidate:
        //成为候选人后立马进行选举
        rf.startElection()

    case Leader:
        rf.electionTimer.Stop()
        rf.heartbeats()
        rf.heartbeatTimer.Reset(HEART_BEAT_TIMEOUT * time.Millisecond)
    }
}
```

适当修改startelection ,收到选票直接记录票 如果够了直接变为leader,高内聚

```go
func (rf *Raft) startElection() {
    ...略 收到选票直接记录票 如果够了直接变为leader
if rf.sendRequestVote(peer, &args, &reply) {
					rf.mu.Lock()
					if reply.VoteGranted && rf.state == Candidate {
						rf.voteCount += 1
						if rf.voteCount > len(rf.peers)/2 {
                            rf.switchStateTo(Leader)
                        }
					} else if reply.Term > rf.currentTerm {
						rf.currentTerm = reply.Term
						rf.switchStateTo(Follower)
					}
					rf.mu.Unlock()
				} else {
					DPrintf("raft%v[%v] vote:raft%v no reply, currentTerm:%v\n", rf.me, rf.state, peer, rf.currentTerm)
				}
    ...略
}
```

2A完成

#### 2B

.我们希望Raft保持一致的、复制日志的操作。在leader处调用Start()将启动向日志添加新操作的过程;leader将新操作发送到附录条目rpc中的其他服务器。

raft 里新增提交通道

```
applyCh        chan ApplyMsg // 提交通道
```

RequestVote 里新增判断， 看leader的日志是否是最新的，不是就不投它了，这是成为leader的必要条件

![image-20211002181520455](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002181520455.png)

```go
//If votedFor is null or candidateId, and candidate’s log is at least as up-to-date as receiver’s log, grant vote
if args.LastLogTerm < rf.log[lastLogIndex].Term ||
        (args.LastLogTerm == rf.log[lastLogIndex].Term &&
            args.LastLogIndex < (lastLogIndex)) {
        // Receiver is more up-to-date, does not grant vote
        reply.Term = rf.currentTerm
        reply.VoteGranted = false
        return
    }
```

![image-20211002181507823](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002181507823.png)

AppendEntries 里新增 如果传入的log的index 好不是自己要的（lastLogIndex < args.PrevLogIndex），则reply false

```go
// 2. Reply false if log doesn’t contain an entry at prevLogIndex
    // whose term matches prevLogTerm (§5.3)
	lastLogIndex := len(rf.log) - 1
    if lastLogIndex < args.PrevLogIndex {
        reply.Success = false
        reply.Term = rf.currentTerm
        return
    }
```

AppendEntries 里继续新增， 如果和已有的冲突了， 那么就删除已经存在的，删除的方式是返回reply.Success = false 即可

```go
// 3. If an existing entry conflicts with a new one (same index
    // but different terms), delete the existing entry and all that
    // follow it (§5.3)
    if rf.log[(args.PrevLogIndex)].Term != args.PrevLogTerm {
        reply.Success = false
        reply.Term = rf.currentTerm
        return
    }
```

AppendEntries 里继续新增，如果有能加进去的，这里分两种情况

+ 第一种直接就rf.log没有当前的logentry,直接加进去
+ 第二种rf.log 有，但是日期不对， 需要一个个对比查找，然后记下开头idx，随后添加

```go
// 4. Append any new entries not already in the log
    // compare from rf.log[args.PrevLogIndex + 1]
	unmatch_idx := -1
	for idx := range args.Entries{
		if len(rf.log) < (args.PrevLogIndex + 2 + idx) || 
			rf.log[(args.PrevLogIndex + 1 + idx)].term != args.Entries[idx].Term {
				unmatch_idx = idx
				break
			}
	}
	if unmatch_idx != -1 {
		// there are unmatch entries
        // truncate unmatch Follower entries, and apply Leader entries
		rf.log = rf.log[:(args.PrevLogIndex + 1 + unmatch_idx)]
		rf.log = append(rf.log, args.Entries[unmatch_idx:]...)
	}
```

添加完成后，更新当前commitIndex

```go
//5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
    if args.LeaderCommit > rf.commitIndex {
        rf.setCommitIndex(min(args.LeaderCommit, len(rf.log)-1))
    }

    reply.Success = true
```



rf.start() 把command 加进去就好了，更新各个参数

```go
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	index := -1
	term := -1
	isLeader := true

	// Your code here (2B).
	rf.mu.Lock() 
	defer rf.mu.Unlock()
	term = rf.currentTerm
	isLeader = rf.state == rf.state.Leader 
	if isLeader {
		rf.log = append(rf.log, LogEntry{Command:command, Term: term})
		index = len(rf.log) - 1
		rf.lastApplied = index
		rf.matchIndex[rf.me] = index
        rf.nextIndex[rf.me] = index + 1
	}

	return index, term, isLeader
}
```

rf.Make()  新增提交通道， log等

```go
	rf.applyCh = applyCh
    rf.log = make([]LogEntry, 1) // start from index 1
	
	rf.nextIndex = make([]int, len(rf.peers))
    rf.matchIndex = make([]int, len(rf.peers))
```

switchStateTo 调整leader的情况 调整所有的nextIndex为 leader last log index + 1 设置成自己的下一个，把matchIndex 设为0

```go
//切换状态，调用者需要加锁
func (rf *Raft) switchStateTo(state int) {
case Leader:
        // initialized to leader last log index + 1
        for i := range rf.nextIndex {
            rf.nextIndex[i] = (len(rf.log))
        }
        for i := range rf.matchIndex {
            rf.matchIndex[i] = 0
        }

        rf.electionTimer.Stop()
        rf.heartbeats()
        rf.heartbeatTimer.Reset(HEART_BEAT_TIMEOUT * time.Millisecond)
}
```

heartbeat 注意， heartbeat 是要把当前entry的前一个LogIndex 和 LogTerm 发过去，根据每个的 nextindex 然后发送对应之后全部的logEntry, 

+ 成功的话，更新对应peer 的 matchIndex 以及 nextIndex，发送成功后更新检查是否大部分 matchIndex 的 发过去了， 如果是 而且对应的 rf.log[N].Term == rf.currentTerm （是自己任期的log）就提交
+ 失败的话，判断失败原因
  + 如果是reply.Term > rf.currentTerm , 说明要变成follower
  + 另一种情况是 发送的index 是对方没有的或者是有但是任期不等的 开始减少对应 nextIndex

```go
// 发送心跳包，调用者需要加锁
func (rf *Raft) heartbeats() {
    for i := range rf.peers {
        if i != rf.me {
            go rf.heartbeat(i)
        }
    }
}

func (rf *Raft) heartbeat(server int) {
	rf.mu.Lock()
    if rf.state != Leader {
        rf.mu.Unlock()
        return
    }
	//下一个之前的那个
	prevLogIndex := rf.nextIndex[server] - 1

	// use deep copy to avoid race condition
    // when override log in AppendEntries()
    entries := make([]LogEntry, len(rf.log[(prevLogIndex+1):]))
    copy(entries, rf.log[(prevLogIndex+1):])

	args := AppendEntriesArgs{
        Term:         rf.currentTerm,
        LeaderId:     rf.me,
        PrevLogIndex: prevLogIndex,
        PrevLogTerm:  rf.log[(prevLogIndex)].Term,
        Entries:      entries,
        LeaderCommit: rf.commitIndex,
    }
    rf.mu.Unlock()

	var reply AppendEntriesReply
	if rf.sendAppendEntries(server, &args, &reply) {
		rf.mu.Lock()
		if rf.state != Leader {
			rf.mu.Unlock()
			return 
		}
		// If last log index ≥ nextIndex for a follower: send
        // AppendEntries RPC with log entries starting at nextIndex
        
        // • If AppendEntries fails because of log inconsistency:
        // decrement nextIndex and retry (§5.3)
		if reply.Success {
			// • If successful: update nextIndex and matchIndex for
        	// follower (§5.3)
			rf.matchIndex[server] = args.PrevLogIndex + len(args.Entries)
            rf.nextIndex[server] = rf.matchIndex[server] + 1

			// If there exists an N such that N > commitIndex, a majority
            // of matchIndex[i] ≥ N, and log[N].term == currentTerm:
            // set commitIndex = N (§5.3, §5.4).
			for N := (len(rf.log) -1); N > rf.commitIndex; N--{
				count := 0
				for _, matchIndex := range rf.matchIndex{
					if matchIndex >= N {
						count += 1
					}
				}
				// 自己已经算进去了 rf.log[N].Term == rf.currentTerm 很重要 之前没加 只能提交自己的
				if count > len(rf.peers) / 2 && rf.log[N].Term == rf.currentTerm{
					// most of nodes agreed on rf.log[i]
                    rf.setCommitIndex(N)
                    break
				}
			}
		} else {
			// • If AppendEntries fails because of log inconsistency:
        	// decrement nextIndex and retry (§5.3)
			if reply.Term > rf.currentTerm {
				rf.currentTerm = reply.Term
                rf.switchStateTo(Follower)
			} else {
				// need to minus the prevlogindex
				rf.nextIndex[server] = args.PrevLogIndex - 1
				// 没有也行  仔细想了想 其实是不能有
				//rf.matchIndex[server] = rf.nextIndex[server] - 1
			}
		}
		rf.mu.Unlock()
		
	}
}
```

随后是  setCommitIndex， 提交 就是把commitidx 与 lastapply 的一个个提交就好了，因为是通道提交， 会有阻塞， 所以再起一个goroutine， （只有自己可以“享”用）

```go
// several set opertion, should be called with a lock
func (rf *Raft) setCommitIndex(commitIndex int) {
	rf.commitIndex = commitIndex
	// apply all entries between lastApplied and committed
    // should be called after commitIndex updated
	if rf.commitIndex > rf.lastApplied {
		DPrintf("%v apply from index %d to %d", rf, rf.lastApplied+1, rf.commitIndex)
        entriesToApply := append([]LogEntry{}, rf.log[(rf.lastApplied+1):(rf.commitIndex+1)]...)
		// 一个个提交 所以起一个 goroutine
		go func(startIdx int, entries []LogEntry) {
			for idx, entry := entries{
				var msg ApplyMsg
				msg.CommandValid = true
                msg.Command = entry.Command
                msg.CommandIndex = startIdx + idx
				rf.applyCh <- msg
				// do not forget to update lastApplied index
                // this is another goroutine, so protect it with lock
                rf.mu.Lock()
                if rf.lastApplied < msg.CommandIndex {
                    rf.lastApplied = msg.CommandIndex
                }
                rf.mu.Unlock()
			}
		}(rf.lastApplied + 1, entriesToApply)
	}
}
```

startElection 几乎没啥变化， 就是把args 的参数多传送了几个， 略

![image-20211002202025556](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002202025556.png)



#### 2C

在每次更改时将Raft的持久状态写入磁盘，并在重新启动时从磁盘读取最新保存的状态。

修改 persist 和 readPersist , 按照要求进行encode 和decode即可，总的来说不能 就是在需要持久化的时候持久化就好了

![image-20211002225627815](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002225627815.png)

```go
e.Encode(rf.currentTerm)
e.Encode(rf.votedFor)
e.Encode(rf.log)
```

修改 RequestVote, 可能会改动votedfor, 其余不变

```go
defer rf.persist() 
```

AppendEntriesReply 新加入处理冲突的元素 见图 锁定冲突的index 和所属 term

![image-20211002234046602](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211002234046602.png)

```go
type AppendEntriesReply struct {
	Term    int  //currentTerm, for leader to update itself
	Success bool //true if follower contained entry matching prevLogIndex and prevLogTerm

	//2C added
	ConflictTerm  int // 2C
    ConflictIndex int // 2C
}
```



同样 AppendEntries 完成后要持久化 因为有可能改变Log[]  另外把产生冲突的log所对应的term 的第一个log的 index 找出来标记上

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
... 略
defer rf.persist() // 改动需要持久化
... 略
// 2. Reply false if log doesn’t contain an entry at prevLogIndex
    // whose term matches prevLogTerm (§5.3)
    lastLogIndex := len(rf.log) - 1
    if lastLogIndex < args.PrevLogIndex {
        reply.Success = false
        reply.Term = rf.currentTerm
        // optimistically thinks receiver's log matches with Leader's as a subset
        reply.ConflictIndex = len(rf.log)
        // no conflict term
        reply.ConflictTerm = -1
        return
    }
    
     // 3. If an existing entry conflicts with a new one (same index
    // but different terms), delete the existing entry and all that
    // follow it (§5.3)
    if rf.log[(args.PrevLogIndex)].Term != args.PrevLogTerm {
        reply.Success = false
        reply.Term = rf.currentTerm
        // receiver's log in certain term unmatches Leader's log
        reply.ConflictTerm = rf.log[args.PrevLogIndex].Term

        // expecting Leader to check the former term
        // so set ConflictIndex to the first one of entries in ConflictTerm
        conflictIndex := args.PrevLogIndex
        // apparently, since rf.log[0] are ensured to match among all servers
        // ConflictIndex must be > 0, safe to minus 1
        for rf.log[conflictIndex-1].Term == reply.ConflictTerm {
            conflictIndex--
        }
        reply.ConflictIndex = conflictIndex
        return
    }
}
```

start() 如果是Leader 也需要持久化

```
if isLeader {
		rf.log = append(rf.log, LogEntry{Command:command, Term: term})
		rf.persist() // 改动需要持久化
		index = len(rf.log) - 1
		rf.matchIndex[rf.me] = index
        rf.nextIndex[rf.me] = index + 1
	}
```

在make() 中, 读取之前持久化的状态，更新nextIndex

```go
// initialize from state persisted before a crash
    rf.mu.Lock()
    rf.readPersist(persister.ReadRaftState())
    rf.mu.Unlock()
	rf.nextIndex = make([]int, len(rf.peers))
    //for persist
    for i := range rf.nextIndex {
        // initialized to leader last log index + 1
        rf.nextIndex[i] = len(rf.log)
    }
    rf.matchIndex = make([]int, len(rf.peers))
```

matchIndex 再看看 应该是只有成功了才更新，也可以把自己的给更新成len(log) - 1

heartbeat 注意，成功的话不做任何更改重点更改失败了的情况

+ 成功的话，更新对应peer 的 matchIndex 以及 nextIndex，发送成功后更新检查是否大部分 matchIndex 的 发过去了， 如果是 而且对应的 rf.log[N].Term == rf.currentTerm （是自己任期的log）就提交
+ 失败的话，判断失败原因
  + 如果是reply.Term > rf.currentTerm , 说明要变成follower，身份改变需要持久化
  + 另一种情况是 发送的index 是对方没有的或者是有但是任期不等的 开始减少对应 nextIndex， 直接用冲突任期当做 nextIndex 了， 其实作者也认为这个优化不是很有必要，毕竟很少丢东西

```go
else {
				// need to minus the prevlogindex
				// rf.nextIndex[server] = args.PrevLogIndex - 1
				rf.nextIndex[server] = reply.ConflictIndex

				 // 感觉非必要
				 // if term found, override it to
                // the first entry after entries in ConflictTerm
				// if reply.ConflictTerm != -1 {
                //     for i := args.PrevLogIndex; i >= 1; i-- {
                //         if rf.log[i-1].Term == reply.ConflictTerm {
                //             // in next trial, check if log entries in ConflictTerm matches
                //             rf.nextIndex[server] = i
                //             break
                //         }
                //     }
                // }
```

startElection 同样需要持久化，votedFor 发生了改变

```
rf.votedFor = rf.me //vote for me
rf.persist()        // 改动需要持久化
```

#### LAB 3

您的键/值服务将是一个复制的状态机，由几个使用Raft来维护复制的键/值服务器组成。只要大多数服务器是活动的，并且可以通信，您的键/值服务就应该继续处理客户机请求，尽管存在其他故障或网络分区。

![image-20211004134126942](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211004134126942.png)

**实现server**

实现的主要想法是通过rf.start发送OP， 随后通过rf.commitindex函数会通过applych进行提交，所以StartKVServer 里面启动一个主线程监听，如果是自己作为leader 发起的，（无论如何都会先验证重复，不重再执行） 那么就有相应的chan 进行传输数据（主要传输clientid 以及 requestid 进行身份验证），否则说明不是自己发起的， 不用管。另外如果是自己发起的（自己是Leader），那么会在发起后等待执行(一起写进一个函数waitApplying)，等待的方式是开一个通道，同时限时，select 一下， 如果通道到了，还需判断是不是自己提交的，（如果挂了，那么就是别人提交的，那么此时clientId 以及 RequestId 一定是不一致的，因为还没给client回信，所以不可能是客户端又发了一次请求），另外为了防止客户端执行两次，验重方式是每次执行前判断此clientId 最后一次执行的lastappliedrequestID, 如果 > lastappliedrequestID , 则验重通过，否则不通过。

OP描述

```go
type Op struct {
	// Your definitions here.
	// Field names must start with capital letters,
	// otherwise RPC will break.
	Key   string
	Value string
	Name  string

	ClientId  int64
	RequestId int
}
```

KVServer

```go
type KVServer struct {
	mu      sync.Mutex
	me      int
	rf      *raft.Raft
	applyCh chan raft.ApplyMsg

	maxraftstate int // snapshot if log grows this big

	// Your definitions here.
	db                   map[string]string         //3A
	dispatcher           map[int]chan Notification //3A
	lastAppliedRequestId map[int64]int             //3A
}
```

通道（判断是否等待执行后返回的是此条命令ID by ClientId & RequestId）

```go
type Notification struct {
	ClientId  int64
	RequestId int
}
```

Get 请求 构造一个get OP 随后发送等待即可 ，如果返回成功直接取值（此时值可能已经改变，但要求不那么严，如果要求严格就在执行后 通知的时候把get value拿出来, 通过Notification 一并传入waitApplying， 随后去除返回）

```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	// Your code here.

	op := Op{
		Key:       args.Key,
		Name:      "Get",
		ClientId:  args.ClientId,
		RequestId: args.RequestId,
	}

	// wait for being applied
	// or leader changed (log is overrided, and never gets applied)
	reply.WrongLeader = kv.waitApplying(op, 500*time.Millisecond)

	if reply.WrongLeader == false {
		kv.mu.Lock()
		value, ok := kv.db[args.Key]
		kv.mu.Unlock()
		if ok {
			reply.Value = value
			return
		}
		// not found
		reply.Err = ErrNoKey
	}
}
```

类似的还有putappend

```go
func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
	// Your code here.
	// Your code here.
    op := Op{
        Key:       args.Key,
        Value:     args.Value,
        Name:      args.Op,
        ClientId:  args.ClientId,
        RequestId: args.RequestId,
    }

    // wait for being applied
    // or leader changed (log is overrided, and never gets applied)
    reply.WrongLeader = kv.waitApplying(op, 500*time.Millisecond)
}

```

startKVServer 由于要求立即返回 所以启动 goroutine 顺序监听，判断是否 重复，如果不重就执行（get 跳过）随后 通过是否有chan 来初步判断是不是作为leader发的，如果有就把clientid & RequestId 传到通道内（由applywating 负责接收）

```go
func StartKVServer(servers []*labrpc.ClientEnd, me int, persister *raft.Persister, maxraftstate int) *KVServer {
	// call labgob.Register on structures you want
	// Go's RPC library to marshall/unmarshall.
	labgob.Register(Op{})

	kv := new(KVServer)
	kv.me = me
	kv.maxraftstate = maxraftstate

	// You may need initialization code here.
	kv.db = make(map[string]string)
	kv.dispatcher = make(map[int]chan Notification)
	kv.lastAppliedRequestId = make(map[int64]int)

	kv.applyCh = make(chan raft.ApplyMsg)
	kv.rf = raft.Make(servers, me, persister, kv.applyCh)

	// You may need initialization code here.
	go func() {
		for msg := range kv.applyCh{
			if msg.CommandValid == false {
				continue
			}

			op := msg.Command.(Op)
			DPrintf("kvserver %d start applying command %s at index %d, request id %d, client id %d",
			kv.me, op.Name, msg.CommandIndex, op.RequestId, op.ClientId)
			kv.mu.Lock()
			if kv.isDuplicateRequest(op.ClientId, op.RequestId) {
				kv.mu.Unlock()
				continue 
			}
			switch op.Name {
			case "Put":
				kv.db[op.key] = op.Value 
			case "Append":
				kv.db[op.Key] += op.Value
                // Get() does not need to modify db, skip
			}
			kv.lastAppliedRequestId[op.ClientId] = op.RequestId

			if ch, ok := kv.dispatcher[msg.CommandIndex]; ok{
				notify := Notification{
					ClientId:  op.ClientId,
                    RequestId: op.RequestId,
				}
				ch <- notify
			}
			kv.mu.Unlock()
            DPrintf("kvserver %d applied command %s at index %d, request id %d, client id %d",
                kv.me, op.Name, msg.CommandIndex, op.RequestId, op.ClientId)
		}
	}()
	return kv
}
```

waitapplying 通过rf.start 提交后 创建一个通道 接下来 select 等待 如果通道传来了消息就看是不是作为leader刚发的，如果不是说明换leader了（换了一个leader commit一个别的），如果时间到期了，就 通过查重判断刚才的是否提交成功了，这里意味着client 不能一下子发好几个不一样的），随后返回

```go
func (kv *KVServer) waitApplying(op Op, timeout time.Duration) bool {
	// return common part of GetReply and PutAppendReply
	// i.e., WrongLeader
	index, _, isLeader := kv.rf.Start(op)
	if isLeader == false {
		return true
	}

	var wrongLeader bool 
	kv.mu.Lock()
	if _, ok := kv.dispatcher[index]; !ok {
		kv.dispatcher[index] = make(chan Notification, 1)
	}
	ch := kv.dispatcher[index]
	kv.mu.Unlock()
	select {
	case notify := <-ch:
		if notify.ClientId != op.ClientId || notify.RequestId != op.RequestId {
            // leader has changed
            wrongLeader = true
        } else {
            wrongLeader = false
        }
		
	case <- time.After(timeout):
		kv.mu.Lock()
		if kv.isDuplicateRequest(op.ClientId, op.RequestId) {
			// 提交成功
			wrongLeader = false
		} else {
			wrongLeader = true 
		}
        //错了 wrong
		//kv.mu.Lock()
        kv.mu.Unlock()
	}
	DPrintf("kvserver %d got %s() RPC, insert op %+v at %d, reply WrongLeader = %v",
        kv.me, op.Name, op, index, wrongLeader)

	kv.mu.Lock()
	delete(kv.dispatcher, index)
	kv.mu.Unlock()
	return wrongLeader
}
```

查重就是根据clientId 判断lastapplidrequestId 号从而进行判断是否重复

```go
func (kv *KVServer) isDuplicateRequest(clientId int64, requestId int) bool {
	appliedRequestId, ok := kv.lastAppliedRequestId[clientId]
	if ok == false || requestId > appliedRequestId {
		return false
	}
	return true
}
```

**Client**

clerk, 就是定义有关结构 远程call 就好了，如果没调用成功就换一台服务器接着发

```go
type Clerk struct {
	servers []*labrpc.ClientEnd
	// You will have to modify this struct.
	leaderId      int //记下来 这样下一次号找leader
	clientId      int64
	lastRequestId int
}

func MakeClerk(servers []*labrpc.ClientEnd) *Clerk {
	ck := new(Clerk)
	ck.servers = servers
	// You'll have to add code here.
	ck.clientId = nrand() // in real world the id can be a unique ip:port
	return ck
}
func (ck *Clerk) Get(key string) string {

	// You will have to modify this function.
	requestId := ck.lastRequestId + 1

	for {
		args := GetArgs{
			Key:       key,
			ClientId:  ck.clientId,
			RequestId: requestId,
		}

		var reply GetReply
		ok := ck.servers[ck.clientId].Call("KVServer.Get", &args, &reply)
		if ok == false || reply.WrongLeader == true {
			ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
			continue
		}
		// request is sent successfully
		ck.lastRequestId = requestId
		return reply.Value
	}
	// 一般执行不到这
	return ""
}

func (ck *Clerk) PutAppend(key string, value string, op string) {
	// You will have to modify this function.
	requestId := ck.lastRequestId + 1
	for {
		args := PutAppendArgs{
			Key:       key,
			Value:     value,
			Op:        op,
			ClientId:  ck.clientId,
			RequestId: requestId,
		}
		var reply PutAppendReply

		ok := ck.servers[ck.leaderId].Call("KVServer.PutAppend", &args, &reply)
		if ok == false || reply.WrongLeader == true {
			ck.leaderId = (ck.leaderId + 1) % len(ck.servers)
			continue
		}
		// request is sent successfully
		ck.lastRequestId = requestId
		return
	}
}

```

common 略

#### 3B 实现快照

通过applymsg 传送snapshot,   只有leader才能发快照

```go
type ApplyMsg struct {
	CommandValid bool
	Command      interface{}
	CommandIndex int

	//to send kv snapshot to KV server
	CommandData []byte // 3b
}
```

Raft 里新增

```go
snapshottedIndex int // 3b snapshot 上次存到哪了
```

persist里也要存这个位置

```
e.Encode(rf.snapshottedIndex)
```

同样 readPersist 的时候

```go
rf.snapshottedIndex = snapshottedIndex
// for lab 3b, we need to set them at the first index
// i.e., 0 if snapshot is disabled
rf.commitIndex = snapshottedIndex
rf.lastApplied = snapshottedIndex
```

另外需要修改的是appendentries 

+ 当传过来的log 的前一个序号还在 当前peer自己的 snapshottedIndex之前 就要进行截取了

```go
if args.PrevLogIndex <= rf.snapshottedIndex {
		reply.Success = true 

		if args.PrevLogIndex + len(args.Entries) > rf.snapshottedIndex {
			startIdx := rf.snapshottedIndex - args.PrevLogIndex
			// only keep the last snapshotted one 
			rf.log = rf.log[:1]
			rf.log = append(rf.log, args.Entries[startIdx:]...)
		}
		return 
	}
```

+ 此外要判断的是 log最后一个index 是否比args中的 prevlogindex 小， 首先先拿到绝对index, 有感lastindex 全部替换， 同样args里的index 要是想和raft中直接对比需要进行相对转换 

  ```go
  func (rf *Raft) getAbsoluteLogIndex(index int) int {
      // index of log including snapshotted ones
      return index + rf.snapshottedIndex
  }
  
  func (rf *Raft) getRelativeLogIndex(index int) int {
      // index of rf.log
      return index - rf.snapshottedIndex
  }
  ```

  heartbeat 改动较大 如果 某个peer 的 prevLogIndex < rf.snapshottedIndex 那么就要直接传snapshot了

  ```go
  if prevLogIndex < rf.snapshottedIndex {
          // leader has discarded log entries the follower needs
          // send snapshot to follower and retry later
          rf.mu.Unlock()
          go rf.syncSnapshotWith(server)
          return
      }
  ```

  实现同步快照 由leader 发送给follower  返回成功更新 matchindex 以及 nextindex

  ```go
  // invoke by Leader to sync snapshot with one follower
  func (rf *Raft) syncSnapshotWith(server int) {
  	rf.mu.Lock()
  	if rf.state != Leader {
          rf.mu.Unlock()
          return
      }
  	args := InstallSnapShotArgs{
  		Term:              rf.currentTerm,
          LeaderId:          rf.me,
          LastIncludedIndex: rf.snapshottedIndex,
          LastIncludedTerm:  rf.log[0].Term,  //keep snapshottedIndex as a guard at rf.log[0]
          Data:              rf.persister.ReadSnapshot(),
  	}
  	DPrintf("%v sync snapshot with server %d for index %d, last snapshotted = %d",
          rf, server, args.LastIncludedIndex, rf.snapshottedIndex)
  	rf.mu.Unlock()
  
  	var reply InstallSnapshotReply
  
  	if rf.sendInstallSnapshot(server, &args, &reply) {
          rf.mu.Lock()
          if reply.Term > rf.currentTerm {
              rf.currentTerm = reply.Term
              rf.switchStateTo(Follower)
              rf.persist()
          } else {
              if rf.matchIndex[server] < args.LastIncludedIndex {
                  rf.matchIndex[server] = args.LastIncludedIndex
              }
              rf.nextIndex[server] = rf.matchIndex[server] + 1
          }
          rf.mu.Unlock()
      }
  }
  ```

  接收快照, 其中6是保留 可取的log（也就是snapshot中的 lastindex 所对应的term 与args 相同），则保留之后的，否则全部删除，仅仅留一个guard log (snapshot中的 lastindex , 主要是为了留它的term)

  ![image-20211004010005027](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211004010005027.png)

  ```go
  func (rf *Raft) sendInstallSnapshot(server int, args *InstallSnapshotArgs, reply *InstallSnapshotReply) bool {
      ok := rf.peers[server].Call("Raft.InstallSnapshot", args, reply)
      return ok
  }
  
  func (rf *Raft) InstallSnapshot(args *InstallSnapshotArgs, reply *InstallSnapshotReply) {
  	rf.mu.Lock()
      defer rf.mu.Unlock()
      // we do not need to call rf.persist() in this function
      // because rf.persister.SaveStateAndSnapshot() is called
  
      reply.Term = rf.currentTerm
      if args.Term < rf.currentTerm {
          return
      }
  
  	//func as heartbeat 因为heartbeat 的时候 如果发现nextIndex < leader 自己的snapshotindex 就直接传snapshot
  	// 所以InstallSnapshot 发挥着 heartbeat的功能
  	rf.heartbeatTimer.Reset(HEART_BEAT_TIMEOUT * time.Millisecond)
  
  	if args.LastIncludedIndex < rf.snapshottedIndex {
  		return 
  	}
  
  	if args.Term > rf.currentTerm {
          rf.currentTerm = args.Term
          rf.switchStateTo(Follower)
          // do not return here.
      }
  
  	// step 2, 3, 4 is skipped because we simplify the "offset"
  
  	// 6. if existing log entry has same index and term with
      //    last log entry in snapshot, retain log entries following it
  	lastIncludedRelativeIndex := rf.getRelativeLogIndex(args.LastIncludedIndex)
  	if len(rf.log) > lastIncludedRelativeIndex &&
          rf.log[lastIncludedRelativeIndex].Term == args.LastIncludedTerm {
          rf.log = rf.log[lastIncludedRelativeIndex:]
      } else {
  		// 7. discard entire log
          rf.log = []LogEntry{{Term: args.LastIncludedTerm, Command: nil}}
  	}
  
  	// 5. save snapshot file, discard any existing snapshot
      rf.snapshottedIndex = args.LastIncludedIndex
  	// IMPORTANT: update commitIndex and lastApplied because after sync snapshot,
      //                         it has at least applied all logs before snapshottedIndex
      if rf.commitIndex < rf.snapshottedIndex {
          rf.commitIndex = rf.snapshottedIndex
      }
      if rf.lastApplied < rf.snapshottedIndex {
          rf.lastApplied = rf.snapshottedIndex
      }
  
  	
  
  	rf.persister.SaveStateAndSnapshot(rf.encodeRaftState(), args.Data)
  
  	if rf.lastApplied > rf.snapshottedIndex {
          // snapshot is elder than kv's db
          // if we install snapshot on kvserver, linearizability will break
          return
      }
  	
  	installSnapshotCommand := ApplyMsg{
          CommandIndex: rf.snapshottedIndex,
          Command:      "InstallSnapshot",
          CommandValid: false,
          CommandData:  rf.persister.ReadSnapshot(),
      }
  	go func(msg ApplyMsg) {
  		rf.applyCh <- msg
  	}(installSnapshotCommand)
  }
  
  // 3B
  type InstallSnapShotArgs struct {
  	// do not need to implement "chunk"
      // remove "offset" and "done"
  	Term				int 
  	LeaderId 			int 
  	lastIncludedIndex	int 
  	lastIncludedTerm	int
  	Data				[]byte
  }
  
  type InstallSnapshotReply struct {
      Term int // 3B
  }
  ```

  **在server.go中**

  KVServer 增加field

  ```go
  appliedRaftLogIndex int // 3B
  ```

  StartKVServer 中增加recover from snapshot 内容 执行完命令后

  ```go
   // 3B: recover from snapshot
      snapshot := persister.ReadSnapshot()
      kv.installSnapshot(snapshot)
      
      // 3B 执行完命令后 可以自己实行这个 比之前有所改进
  			if kv.shouldTakeSnapshot() {
  				kv.takeSnapshot()
  			}
  ```

  ```go
  // 3B
  func (kv *KVServer) installSnapshot(snapshot []byte) {
      kv.mu.Lock()
      defer kv.mu.Unlock()
      if snapshot != nil {
          r := bytes.NewBuffer(snapshot)
          d := labgob.NewDecoder(r)
          if d.Decode(&kv.db) != nil ||
              d.Decode(&kv.lastAppliedRequestId) != nil {
              DPrintf("kvserver %d fails to recover from snapshot", kv.me)
          }
      }
  }
  ```

  StartKVServer 中负责监听的go程 增加

  ```go
  if msg.CommandValid == false {
                  //3B
                  switch msg.Command.(string) {
                  case "InstallSnapshot":
                      kv.installSnapshot(msg.CommandData)
                  }
                  continue
   }
  ... 略
  // 而且每apply 一次 command 就
  // 3B
  kv.appliedRaftLogIndex = msg.CommandIndex
  ```

  在waitApplying中, 每次rf.start() 之后

  ```go
  // 3B
  if kv.shouldTakeSnapshot() {
      kv.takeSnapshot()
  }
  ```

  其中

  ```go
  // 3B
  func (kv *KVServer) shouldTakeSnapshot() bool {
  	if kv.maxraftstate == -1 {
  		return false
  	}
  
  	if kv.rf.GetRaftStateSize() >= kv.maxraftstate {
  		return true
  	}
  	return false
  }
  ```

  ```
  func (kv *KVServer) takeSnapshot() {
  	w := new(bytes.Buffer)
  	e := labgob.NewEncoder(w)
  	kv.mu.Lock()
  	e.Encode(kv.db)
  	e.Encode(kv.lastAppliedRequestId)
  	appliedRaftLogIndex := kv.appliedRaftLogIndex
  	kv.mu.Unlock()
  	//清楚之前的 log
  	kv.rf.ReplaceLogWithSnapshot(appliedRaftLogIndex, w.Bytes())
  }
  ```

  在raft.go中

  ```go
  //3B
  func (rf *Raft) ReplaceLogWithSnapshot(appliedIndex int, kvSnapshot []byte) {
      rf.mu.Lock()
      defer rf.mu.Unlock()
      if appliedIndex <= rf.snapshottedIndex {
          return
      }
      // truncate log, keep snapshottedIndex as a guard at rf.log[0]
      // because it must be committed and applied
      rf.log = rf.log[rf.getRelativeLogIndex(appliedIndex):]
      rf.snapshottedIndex = appliedIndex
      rf.persister.SaveStateAndSnapshot(rf.encodeRaftState(), kvSnapshot)
  
      // // update for other nodes 这个 只有leader 能干 //太花时间了 费带宽 不如每个自己干
      // for i := range rf.peers {
      //     if i == rf.me {
      //         continue
      //     }
      //     go rf.syncSnapshotWith(i)
      // }
  }
  ```

## LAB 4

  实现shards ， 这个shards存储系统由两部分组成， 一部分各个replica groups， 每个group 负责一系列的shards，这个组通过raft 保持一致性，另一个组成部分 shard master， 负责决定each shard 由哪个 replica group 负责存储， client 会先问问 Master 找到合适的组，replica group也会问master去决定shards 去服务client.

  这个sharded storage system 必须能够移动shards 在这些replica groups。为了负载均衡，也是为了有的组可以离开或者新的组加入。

  很基础的实现，很多功能没有实现（（1) shards 之间的传递很慢并且不允许 concurrent client acess；2) 每个 raft group 中的 member 不会改变）

  ![image-20211004160226441](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211004160226441.png)

  

   Start with a stripped-down copy of your kvraft server.

  #### 4A

  **configuration service** 

    1. 由若干 shardmaster 利用 raft 协议保证一致性的集群；
    2. 管理 configurations 的顺序：每个 configuration 描述 replica group 以及每个 group 分别负责存储哪些 shards；
    3. 响应 Join/Leave/Move/Query 请求，并且对 configuration 做出相应的改变；

  **replica group**

    1. 由若干 shardkv 利用 raft 协议保证一致性的集群；
    2. 负责具体数据的存储（一部分），组合所有 group 的数据即为整个 database 的数据；
    3. 响应对应 shards 的 Get/PutAppend 请求，并保证 linearized；
    4. 周期性向 shardmaster 进行 query 获取 configuration，并且进行 migration 和 update；

  

  Sharemaster 主要负责根据 Client 提供的分区规则，将数据储存在不同的replica  group 中

  Sharemaster 有多台机器组成，他们之间使用 Raft 协议来保证一致性。

  每一个 replica  group由多台机器组成，他们之间也是通过 Raft 协议来保证一致性

  

  ```go
  // The number of shards.
  const NShards = 10
  
  // A configuration -- an assignment of shards to groups.
  // Please don't change this.
  type Config struct {
  	Num    int              // config number
  	Shards [NShards]int     // shard -> gid
  	Groups map[int][]string // gid -> servers[]
  }
  ```

  一个CONFIG 里面包含了这个CONFIG 的版本号。哪个分区归哪个REPLICA GROUP 管
   这个REPLICA GROUP 里面包含了哪些SERVER
   然后MASTER SERVER 会有一组CONFIG，序号递增。

  总体和3A很相似

  #### 4B

  构造一个KVShard

  先完成任务一， 实现一个和lab3 相似的server StartServer  与之前一样，开一个Go 程接收kv.applyCh 的 applyMsg(初始化阶段省略)

  ```go
  go func() {
          for {
              select {
              case <- kv.killCh:
                  return
              case applyMsg := <- kv.applyCh:
                  if !applyMsg.CommandValid {
                      kv.readSnapShot(applyMsg.SnapShot)
                      continue
                  }
                  kv.apply(applyMsg)
              }
          }
      }()
  ```

  另外，接到客户端请求的时候 依然是通过 rf.start()发送，等待成功提交后（由主线程生成的go程），会发送OP进行确认，然后再给客户端回送，与之前相似，略

  ```go
  func (kv *ShardKV) Get(args *GetArgs, reply *GetReply) {
  	// Your code here.
  	originOp := Op{"Get", args.Key, "", Nrand(), 0}
  	reply.Err, reply.Value = kv.templateStart(originOp)
  }
  
  func (kv *ShardKV) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
  	// Your code here.
  	originOp := Op{args.Op, args.Key, args.Value, args.Cid, args.SeqNum}
  	reply.Err, _ = kv.templateStart(originOp)
  
  }
  
  func (kv *ShardKV) templateStart(originOp Op) (Err, string) {
  	index, _, isLeader := kv.rf.Start(originOp)
  	if isLeader {
  		ch := kv.put(index, true)
  		op := kv.beNotified(ch, index)
  		if equalOp(originOp, op) {
  			return OK, op.Value
  		}
  		if op.OpType == ErrWrongGroup {
  			return ErrWrongGroup, ""
  		}
  	}
  	return ErrWrongLeader, ""
  }
  
  func (kv *ShardKV) put(idx int, createIfNotExists bool) chan Op {
  	kv.mu.Lock()
  	defer kv.mu.Unlock()
  	if ch, ok := kv.chMap[idx]; ok {
  		if !createIfNotExists {
  			return nil
  		}
  		kv.chMap[idx] = make(chan Op, 1)
  	}
  	return kv.chMap[idx]
  }
  
  func (kc *ShardKV) beNotified(ch chan Op, index int) {
  	select {
  	case notifyArg, ok := <-ch:
  		if ok {
  			close(ch)
  		}
  		kv.mu.Lock()
  		delete(kv.chMap, index)
  		kv.mu.Unlock()
  		return notifyArg
  	case <-time.After(time.Duration(1000) * time.Millisecond):
  		return Op{}
  	}
  }
  ```

-- --

  

获取最新config 以及根据config 拒绝不属于自己的shard的代码 from (Add code to `server.go` to periodically fetch 		the latest configuration from the shardmaster, and add 		code to reject client requests if the receiving group 		isn't responsible for the client's key's shard. You 		should still pass the first test. hint1)

在startServer的时候 一开始就试图获得新的config

  ```go
go kv.daemon(kv.tryPollNewCfg,50)
  ```

其中daemon 定期执行函数

```go
func (kv *ShardKV) daemon(do func(), sleepMS int) {
    for {
        select {
        case <-kv.killCh:
            return
        default:
            do()
        }
        time.Sleep(time.Duration(sleepMS) * time.Millisecond)
    }
}
```

获取最新的cfg,根据自己当前的cfg 试图获取更新的，如果成功获得了更新的，就raft同步

```go
func (kv *ShardKV) tryPollNewCfg() {
    _, isLeader := kv.rf.GetState();
    kv.mu.Lock()
    if !isLeader || len(kv.comeInShards) > 0{
        kv.mu.Unlock()
        return
    }
    next := kv.cfg.Num + 1
    kv.mu.Unlock()
    cfg := kv.mck.Query(next)
    if cfg.Num == next {
        kv.rf.Start(cfg) //sync follower with new cfg
    }
}
```

另外 在apply中实现

```go
if cfg, ok := applyMsg.Command.(shardmaster.Config); ok {
        kv.updateInAndOutDataShard(cfg)
    }

```

![image-20211007163907613](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20211007163907613.png)

更新config 从而找出 一部分是需要获得的， 另一部分是需要给出的，提前准备好

```go
func (kv *ShardKV) updateInAndOutDataShard(cfg shardmaster.Config) {
	kv.mu.Lock()
    defer kv.mu.Unlock()
	// seqNO less than the current cfg
	if cfg.Num <= kv.cfg.Num {
		return 
	}
	oldCfg, OldShards := kv.cfg, kv.mvShards
	kv.myShards, kv.cfg = make(map[int]bool), cfg
	for shard, gid := range cfg.Shards {
		if gid != kv.gid { return }
		if _, ok := OldShards[shard]; ok || oldCfg.Num == 0 {
			// the shard dont changed and still keep it
			kv.myShards[shard] = true
            delete(OldShards, shard)
		} else {
			// config is newer so take the shard from the old one
			kv.comeInShards[shard] = oldCfg.Num 
		}
	}
	toOutShards := OldShards
	if len(toOutShards) > 0{
		//prepare the data for migration
		kv.toOutShards[oldCfg.Num] = make(map[int]map[string]string)
		for shard := toOutShards {
			outDb := make(map[string]string)
			for k, v := range kv.db {
				if key2shard(k) == v: {
					outDb[k] = v
					delete(kv.db, k)
				}
			}
			kv.toOutShards[oldCfg.Num][shard] = outDb
		}
	}
}
```



发送SHARD的思考

如果config 发生变化之后， A server 要向B server 去要还是B server 主动发出来，如果是B server 主动发，而C没有接收最新的config就不好 协调了，还是A 要比较好。要的时候肯定带 shard 以及config，便于B 判断，B返回的时候不但要带上所属的 DB（仅包含shard）以及当前shard,还有就是为了防止B 刚刚更新 DB 但还没来及返回给客户端，那么也要把当前客户端所包含的最近的requestSeq发过去

```go
type MigrateArgs struct {
    Shard     int
    ConfigNum int
}

type MigrateReply struct {
    Err         Err
    ConfigNum   int
    Shard       int
    DB          map[string]string
    Cid2Seq     map[int64]int
}
```

StartServer 开始定时检查有没有需要从别的地方pull的。

```go
go kv.daemon(kv.tryPullShard,80)
```

根据comeInShards 提供的shard 以及对应的 config idx 获取对应的config， 然后根据config 中找到哪个组还有所需的shard，对该组一一发送即可

```go
func (kv *ShardKV) tryPullShard() {
	_, isLeade := kv.rf.GetState()
	kv.mu.Lock()
    if  !isLeader || len(kv.comeInShards) == 0 {
        kv.mu.Unlock()
        return
    }
	var wait sync.WaitGroup
	// idx is config NO
	for shard, idx := range kv.comeInShards {
		wait.Add(1)
		go func(shard int, cfg shardmaster.Config){
			defer wair.Done()
			args := MigrateArgs{shard, cfg.Num}
			gid := cfg.Shards[shard]
			for _, server := range cfg.Groups[gid] {
                srv := kv.make_end(server)
                reply := MigrateReply{}
                if ok := srv.Call("ShardKV.ShardMigration", &args, &reply); ok && reply.Err == OK {
                    kv.rf.Start(reply)
                }
            }
		}(shard, kv.mck.Query(idx))

	}
	kv.mu.Unlock()
    wait.Wait()

}
```

leader发送 ShardMigration RPC 索要 db[shard], 每个raft group都由 Leader负责发送和接受RPC， 实现点对点，以点带面，否则就乱了。

为了防止config 来回重新指定 shard， 比如让 A server 接收shard 1(from XX),接下来 不接受， 然后又变得接收，那么只需要接收 在这段时间变化了的 shard1 就行。但为了实现这个，就要保存每个config指定下的shard,

注意，args.configNum < kv.cfg.Num才可以

```go
func (kv *ShardKV) ShardMigration(args *MigrateArgs, reply *MigrateReply) {
	reply.Err, reply.Shard, reply.ConfigNum = ErrWrongLeader, args.Shard, args.ConfigNum
	if _,isLeader := kv.rf.GetState(); !isLeader {return}
    kv.mu.Lock()
    defer kv.mu.Unlock()
	reply.Err = ErrWrongGroup
	// args.confignum == oldconfig.num
	if args.ConfigNum >= kv.cfg.Num {return}
	reply.Err,reply.ConfigNum, reply.Shard = OK, args.ConfigNum, args.Shard
	reply.DB, reply.Cid2Seq = kv.deepCopyDBAndDedupMap(args.ConfigNum,args.Shard)
}
func (kv *ShardKV) deepCopyDBAndDedupMap(config int,shard int) (map[string]string, map[int64]int) {
    db2 := make(map[string]string)
    cid2Seq2 := make(map[int64]int)
    for k, v := range kv.toOutShards[config][shard] {
        db2[k] = v
    }
    for k, v := range kv.cid2Seq {
        cid2Seq2[k] = v
    }
    return db2, cid2Seq2
}
```

同样 raft 开始提交migration 的applymsg 后，在apply进行处理

```go
else if migrationData, ok := applyMsg.Command.(MigrateReply); ok{
        kv.updateDBWithMigrateData(migrationData)
    }
```

如果发现 len(kv.comeInShards) > 0 就不会去拉新的 config 知道全部获得 shard,才会去拉，（key point）我们不得不CHECK 就是当前REPLY的CONFIG版本号必须是当前CONFIG版本号小一个.

```go
func (kv *ShardKV) updateDBWithMigrateData(migrationData MigrateReply) {
	kv.mu.Lock()
    defer kv.mu.Unlock()
	if migrationData.ConfigNum != kv.cfg.Num-1 {return}
	delete(kv.comeInShards, migrationData.Shard)
	 //this check is necessary, to avoid use  kv.cfg.Num-1 to update kv.cfg.Num's shard
	if _, ok := kv.myShards[migrationData.Shard]; !ok {
		kv.myShards[migrationData.Shard] = true
		for k, v := range migrationData.DB {
            kv.db[k] = v
        }
        for k, v := range migrationData.Cid2Seq {
            kv.cid2Seq[k] = Max(v,kv.cid2Seq[k])
        }
		
	}
}	
```

**分析返回wrong group的时机**

如果一上来就判断，而不是传送raft，有可能传送完raft以后，已经变了，所以要在apply里(更准确地说是normal)进行判断。（其实如果变动不频繁的话可以双重判断） 在normal里就要加锁准备好。防止等到OP的时候再去db里拿，而此时db里已经没了（因为起了一个go程 去pull最新的config, 发现变化会删除）

```go
op := applyMsg.Command.(Op)
		if ...
		else {
            kv.normal(&op)
        }
		if notifyCh := kv.put(applyMsg.CommandIndex,false); notifyCh != nil {
            send(notifyCh,op)
        }
```

```go
func (kv *ShardKV) normal(op *Op) {
	shard := key2shard(op.Key)
	kv.mu.Lock()
	if _, ok := kv.myShards[shard]; !ok {
        op.OpType = ErrWrongGroup
    } else {
        maxSeq,found := kv.cid2Seq[op.Cid]
        if !found || op.SeqNum > maxSeq {
            if op.OpType == "Put" {
                kv.db[op.Key] = op.Value
            } else if op.OpType == "Append" {
                kv.db[op.Key] += op.Value
            }
            kv.cid2Seq[op.Cid] = op.SeqNum
        }
        if op.OpType == "Get" {
            op.Value = kv.db[op.Key]
        }
    }
    kv.mu.Unlock()
}
```

有时容易出现交叉死锁，如果某个过程需要两把锁，尽量先拿容易被别人拿的那把

**实现snapshot**

在apply的最后，判断是否需要snapshot, 如果需要就进行snapshot

```go
if kv.needSnapShot() {
        go kv.doSnapShot(applyMsg.CommandIndex)
 }
```

```go
func (kv *ShardKV) needSnapShot() bool {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    // threshold := 10
    return kv.maxraftstate > 0 &&
        kv.maxraftstate < kv.persist.RaftStateSize()
}
```

就是多保存一些状态进行存储

```go
func (kv *ShardKV) doSnapShot(index int) {
    w := new(bytes.Buffer)
    e := labgob.NewEncoder(w)
    kv.mu.Lock()
    e.Encode(kv.db)
    e.Encode(kv.cid2Seq)
    e.Encode(kv.comeInShards)
    e.Encode(kv.toOutShards)
    e.Encode(kv.myShards)
    e.Encode(kv.cfg)
    e.Encode(kv.garbages)
    kv.mu.Unlock()
    kv.rf.DoSnapShot(index,w.Bytes())
}
```

StartServer的时候

```go
kv.readSnapShot(kv.persist.ReadSnapshot())
```

**开始chanllenge 1 (2已经通过)**

startserver  新增一个garbages 属性， 还是开一个go程， 定期从这里向之前数据的提供发消息，待删除后，对方回过来删除成功，就delete掉map里的该项

```
garbages     map[int]map[int]bool              "cfg number -> shards"
```

那么，向garbages添加的时机呢，在apply更新完成某个shard后即可。

```go
if _, ok := kv.garbages[migrationData.ConfigNum]; !ok {
            kv.garbages[migrationData.ConfigNum] = make(map[int]bool)
        }
        kv.garbages[migrationData.ConfigNum][migrationData.Shard] = true
```

在startserver 里开个Go程，

```
go kv.daemon(kv.tryGC,100)
```

其中

```go
func (kv *ShardKV) tryGC() {
	_, isLeader := kv.rf.GetState();
    kv.mu.Lock()
    if !isLeader || len(kv.garbages) == 0{
        kv.mu.Unlock()
        return
    }
	var wait sync.WaitGroup
	for cfgNum, shards := range kv.garbages {
		for shard := range shards {
			wait.add(1)
			go func(shard int, cfg shardmaster.Config){
				defer wait.Done()
				args := MigrateArgs{shard, cfg.Num}
                gid := cfg.Shards[shard]
				for _, server := range cfg.Groups[gid] {
					srv := kv.make_end(server)
					reply := MigrateReply{}
					if ok := srv.Call("ShardKV.GarbageCollection", &args, &reply); ok && reply.Err == OK {
                        kv.mu.Lock()
                        defer kv.mu.Unlock()
                        delete(kv.garbages[cfgNum], shard)
                        if len(kv.garbages[cfgNum]) == 0 {
                            delete(kv.garbages, cfgNum)
                        }
                    }
				}
			}(shard, kv.mck.Query(cfgNum))
		}
	}
	kv.mu.Unlock()
    wait.Wait()
}
```

RPC 接收到以后， 通过raft 发送 Start()

```
func (kv *ShardKV) GarbageCollection(args *MigrateArgs, reply *MigrateReply) {
    reply.Err = ErrWrongLeader
    if _, isLeader := kv.rf.GetState(); !isLeader {return}
    kv.mu.Lock()
    defer kv.mu.Unlock()
    if _,ok := kv.toOutShards[args.ConfigNum]; !ok {return}
    if _,ok := kv.toOutShards[args.ConfigNum][args.Shard]; !ok {return}
    originOp := Op{"GC",strconv.Itoa(args.ConfigNum),"",Nrand(),args.Shard}
    kv.mu.Unlock()
    reply.Err,_ = kv.templateStart(originOp)
    kv.mu.Lock()
}
```

apply里面, 添加对GC的处理

```go
op := applyMsg.Command.(Op)
        if op.OpType == "GC" {
            cfgNum,_ := strconv.Atoi(op.Key)
            kv.gc(cfgNum,op.SeqNum);
} 
```

```go
func (kv *ShardKV) gc(cfgNum int, shard int) {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    if _, ok := kv.toOutShards[cfgNum]; ok {
        delete(kv.toOutShards[cfgNum], shard)
        if len(kv.toOutShards[cfgNum]) == 0 {
            delete(kv.toOutShards, cfgNum)
        }
    }
}
```

完结！



![image-20220307015319338](C:\Users\12756\AppData\Roaming\Typora\typora-user-images\image-20220307015319338.png)
