### outcluster_syn类调用方法

#### 调用方法及结果示例

在主函数中，通过构造函数 tpsn(NodeContainer adhocnodes, uint32_t numofnode, Ipv4InterfaceContainer AdHocIp_temp, vector<int> headlist, vector<double> timeshift, double starttime, int times_temp)创建类的实例。

第一个参数：NodeContainer adhocnodes 传入所有的节点。第二个参数 uint32_t numofnode 总的节点数量。第三个参数 Ipv4InterfaceContainer AdHocIp_temp 所有节点所对应IP地址。第四个参数 vector<int> headlist 为根节点和分簇后的簇头节点的节点序号 。第五个参数vector<double> timeshift 为所有节点的节点偏移。第六个参数 double starttime 为开始进行簇间时间同步的起始时间，第七个参数 int times_temp 为大循环轮数。

在分簇完成之后，通过调用vector<double> tpsn_main(tpsn* ptrtpsn);函数即可完成簇间时间同步，传入参数是实例化的类指针。

**运行示例**

<img src="C:\Users\9613\Desktop\效果图.png" alt="效果图" style="zoom: 67%;" />

**仿真参数**

| 仿真参数   | 数值                                                       |
| ---------- | :--------------------------------------------------------- |
| 节点个数   | 20                                                         |
| 簇头节点   | 9、10、11、14、16                                          |
| 根节点     | 0                                                          |
| 根节点IP   | 195.1.1.1                                                  |
| 簇头节点IP | 195.1.1.10、195.1.1.11、195.1.1.12、195.1.1.15、195.1.1.17 |
| 同步轮数   | 5                                                          |

**输出结果说明**

1. 根节点和簇头节点通过TPSN算法实现时间同步，簇头节点接收到根节点发送的sync_ack信号后，会将接收到的信号内容输出，并结合算法算出delta，并输出使用delta对误差原始误差修正之后的时间误差、同步的轮数。

```
10 receive : '10,4032213.385201,4020026.634125,4030026.634125,0' from 195.1.1.1
delta = -12222.05
误差：0.80
原始误差：12221.25
第 0 轮
```

2. 在最后一轮同步时，簇头节点除了输出上述输出外，还会额外输出同步之后的节点平均误差、以及该簇头经过多次时间同步之后的最终误差。

```
10 receive : '10,4112496.183283,4100308.688021,4110308.688021,4' from 195.1.1.1
delta = -12221.30
误差：0.05
原始误差：12221.25
第 4 轮
节点平均误差: 0.16

簇头节点10的同步后误差-0.05
```

#### 类成员函数和成员变量

1. 类成员变量

| 变量名       | 类型                   | 功能                             |
| ------------ | ---------------------- | -------------------------------- |
| numofnodes   | int32_t                | 簇内节点的总数（包括簇头）       |
| serialNumMax | int                    | 同步轮数                         |
| headList     | vector<int>            | 分簇后的簇头节点序号             |
| t1           | vector<vector<double>> | TPSN时间同步T1时间戳             |
| t2           | vector<vector<double>> | TPSN时间同步T2时间戳             |
| t3           | vector<vector<double>> | TPSN时间同步T3时间戳             |
| t4           | vector<vector<double>> | TPSN时间同步T4时间戳             |
| deltaTotal   | vector<double>         | 累计所有节点的同步误差           |
| total_error  | vector<double>         | 累计单个节点每轮的同步误差       |
| delay        | vector<vector<double>> | TPSN时间同步的sync_req的传播延迟 |
| waiting      | vector<vector<double>> | 每个节点每轮同步的等待时间       |
| StartTime    | vector<vector<double>> | 每轮同步的开始时间               |
| delay2       | vector<vector<double>> | TPSN时间同步的sync_ack的传播延迟 |
| TimeShift    | vector<double>         | 每个节点的时间偏移               |
| AdHocNode    | NodeContainer          | 所有的节点                       |
| AdHocIp      | Ipv4InterfaceContainer | 所有节点所对应的IP地址           |
| nAdHoc       | uint32_t               | 所有的节点数量                   |
| times        | int                    | 大循环轮数                       |

2. 成员函数

| 函数名称            | 输入                                               | 返回值                 | 功能                               |
| ------------------- | -------------------------------------------------- | ---------------------- | ---------------------------------- |
| randnum             | double mean, double var, int num                   | vector<double>         | 产生随机值                         |
| randnumdelay        | double mean, double var, int num, int maxserialnum | vector<vector<double>> | 生成二维随机数                     |
| getTimeShift        | 空                                                 | double                 | 生成时间偏移                       |
| getCurrentLocalTime | double timeShift                                   | string                 | 获取当前时间                       |
| RecvString1         | Ptr<Socket> sock                                   | void                   | sync_req信号的回调函数             |
| RecvString2         | Ptr<Socket> sock                                   | void                   | sync_ack信号的回调函数             |
| send                | Ptr<Socket> sock, int ID_den, int serialnum        | void                   | 信号发送函数                       |
| loss                | 空                                                 | double                 | 丢包率随即数生成函数               |
| tpsn_main           | tpsn* ptrtpsn                                      | vector<double>         | TPSN时间同步函数                   |
| getHeaderT4         | 空                                                 | vector<double>         | 获取TPSN算法最后一个节点的T4时间戳 |

#### TPSN算法同步流程

##### 1. 创建子节点向根节点发送同步消息的连接（子节点向根节点发送T1时间戳）

```C++
    TypeId tid = TypeId::LookupByName("ns3::UdpSocketFactory");
    Ptr<Socket> Recv_sock = Socket::CreateSocket(AdHocNode.Get(0), tid);
    // InetSocketAddress addr = InetSocketAddress(Ipv4Address::GetAny(), 10000);
    InetSocketAddress addr = InetSocketAddress(AdHocIp.GetAddress(0), 10000);
    Recv_sock->Bind(addr);
    Recv_sock->SetRecvCallback(MakeCallback(&tpsn::RecvString1, ptrtpsn)); // 设置回调函数
    // 发送端，所有节点向根节点发送t1
    Ptr<Socket> Send_sock[headList.size()];
    InetSocketAddress RecvAddr = InetSocketAddress(AdHocIp.GetAddress(0), 10000 + times);
    for (uint32_t i = 1; i < (uint32_t)headList.size(); i++)
    {
        Send_sock[i] = Socket::CreateSocket(AdHocNode.Get(headList[i]), tid);
        Send_sock[i]->Connect(RecvAddr);
    }
```

##### 2. 创建根节点向子节点发送同步消息的连接（根节点向子节点发送T1、T2、T3时间戳）

```C++
 Ptr<Socket> Recv_sock_child[headList.size()];
    // InetSocketAddress addr_child = InetSocketAddress(AdHocIp.GetAddress(0), 10000);
    for (uint32_t i = 1; i < (uint32_t)headList.size(); i++)
    {
        InetSocketAddress addr_child = InetSocketAddress(AdHocIp.GetAddress(headList[i]), 10000 + times);
        Recv_sock_child[i] = Socket::CreateSocket(AdHocNode.Get(headList[i]), tid);
        Recv_sock_child[i]->Bind(addr_child);
        Recv_sock_child[i]->SetRecvCallback(MakeCallback(&tpsn::RecvString2, ptrtpsn)); // 设置回调函数
    }
    Ptr<Socket> Send_sock_root[headList.size()];
    for (uint32_t i = 1; i < (uint32_t)headList.size(); i++)
    {
        Send_sock_root[i] = Socket::CreateSocket(AdHocNode.Get(0), tid);
        InetSocketAddress RecvAddr_root = InetSocketAddress(AdHocIp.GetAddress(headList[i]), 10000 + times);
        Send_sock_root[i]->Connect(RecvAddr_root);
    }
```

##### 3. 进行多轮TPSN时间同步

```C++
for (serialnum = 0; serialnum < serialNumMax; serialnum++)
    {
        // synv_req过程
        for (uint32_t j = 1; j < (uint32_t)headList.size(); j++)
        //for (uint32_t i = 1; i < nAdHoc; i++)
        {
            int i = headList[j];
            // 子节点1第一次传输，包内容为T1
            if (serialnum == 0)
            {
                t1[i][serialnum] = StartTime[i][serialnum] + TimeShift[i];
            }
            else
            {
                t1[i][serialnum] = t4[i][serialnum - 1] + waiting[i][serialnum - 1];
                
            }
            t4[i][serialnum] = t1[i][serialnum] + delay[i][serialnum] + delay2[i][serialnum] + waiting[i][serialnum];
            t2[i][serialnum] = delay[i][serialnum] + t1[i][serialnum] - TimeShift[i];
            t3[i][serialnum] = t2[i][serialnum] + waiting[i][serialnum];
            double deltaMain = ((t2[i][serialnum] - t1[i][serialnum]) - (t4[i][serialnum] - t3[i][serialnum])) / 2;
            deltaTotalMain[j] += deltaMain;
            // 有0.01概率丢包，*丢包行为发生后这一轮不同步也不计算误差
            if (loss() < 0.99) {
                Simulator::Schedule(MicroSeconds(t1[i][serialnum] - TimeShift[i] - Simulator::Now().GetMicroSeconds()), &tpsn::send, ptrtpsn, Send_sock[j], 0, serialnum);
                Simulator::Schedule(MicroSeconds(t1[i][serialnum] - TimeShift[i] + delay[i][serialnum] + waiting[i][serialnum] - Simulator::Now().GetMicroSeconds()), &tpsn::send, ptrtpsn, Send_sock_root[j], i, serialnum);
            }
            // 是否需要计算输出上一轮的误差

        }
    }
```

##### 4. 消息发送函数

```c++
void tpsn::send(Ptr<Socket> sock, int ID_den, int serialnum) // 判断T1，T2，T3不存在则为0
{
    //cout << "真正的传输时间：" << Simulator::Now().GetMicroSeconds() << endl;
    uint8_t buffer[255];
    int ID_ori = sock->GetNode()->GetId();
    int ID;
    if (ID_den == 0)
    {
        ID = ID_ori;
    }
    else
    {
        ID = ID_den;
    }
    // 此处ID为被同步节点ID，不论根节点收还是发
    string ID_str = to_string(ID);
    string T1 = to_string(t1[ID][serialnum]);
    string T2 = to_string(t2[ID][serialnum]);
    string T3 = to_string(t3[ID][serialnum]);
    string serialnum_str = to_string(serialnum);
    // cout << "T2 T3: " << T2 << "  " << T3 << endl;
    stringstream ss;
    ss << ID_str << "," << T1 << "," << T2 << "," << T3 << "," << serialnum_str;
    string resultString = ss.str();
    uint32_t len = resultString.length();
    for (uint32_t i = 0; i < len; i++)
    {
        buffer[i] = resultString[i]; // char 与 uint_8逐个赋值
    }
    buffer[len] = '\0';
    Ptr<Packet> p = Create<Packet>(buffer, sizeof(buffer)); // 把buffer写入到包内
    // cout << "ID " << ID_ori << " Send to " << ID_den << " : " << buffer << endl;
    sock->Send(p);
}

```

##### 5. sync_req的消息响应函数

```C++
void tpsn::RecvString1(Ptr<Socket> sock) // 回调函数
{

    Address from;
    Ptr<Packet> packet = sock->RecvFrom(from);
    packet->RemoveAllPacketTags();
    packet->RemoveAllByteTags();
    InetSocketAddress address = InetSocketAddress::ConvertFrom(from);

    // uint8_t data[sizeof(packet)];
    uint8_t data[255];
    packet->CopyData(data, sizeof(data)); // 将包内数据写入到data内
    cout << sock->GetNode()->GetId() << " "
         << "receive : '" << data << "' from " << address.GetIpv4() << endl;

    // 解析data
    char a[sizeof(data)];
    for (uint32_t i = 0; i < sizeof(data); i++)
    {
        a[i] = data[i];
    }
    string strres = string(a);
    stringstream ss(strres);
    vector<string> resultStrings;
    string token;

    while (getline(ss, token, ','))
    {
        resultStrings.push_back(token);
    }
    int ID = stoi(resultStrings[0]);
    int serialnum = stod(resultStrings[4]);
    t1[ID][serialnum] = stod(resultStrings[1]);
    t2[ID][serialnum] = stod(resultStrings[2]);
    t3[ID][serialnum] = stod(resultStrings[3]);
    cout << "从" << ID << "接收到的数据为："
         << "T1 = " << t1[ID][serialnum] << ", T2 = " << t2[ID][serialnum] << ", T3 = " << t3[ID][serialnum] << endl;

    // 把接收到的数据处理后再发回去
    // string localtime = getCurrentLocalTime(0);
    t2[ID][serialnum] = delay[ID][serialnum] + t1[ID][serialnum] - TimeShift[ID];
    t3[ID][serialnum] = t2[ID][serialnum] + waiting[ID][serialnum];

    cout << "即将发送的数据为："
         << "T1 = " << t1[ID][serialnum] << ", T2 = " << t2[ID][serialnum] << "     " << t1[ID][serialnum] - TimeShift[ID] << "    " << delay[ID][serialnum] << "  , T3 = " << t3[ID][serialnum] << endl;
    cout << endl;
}
```

##### 6.sync_ack的消息响应函数

```C++
void tpsn::RecvString2(Ptr<Socket> sock) // 回调函数
{

    Address from;
    Ptr<Packet> packet = sock->RecvFrom(from);
    packet->RemoveAllPacketTags();
    packet->RemoveAllByteTags();
    InetSocketAddress address = InetSocketAddress::ConvertFrom(from);

    // uint8_t data[sizeof(packet)];
    uint8_t data[255];
    packet->CopyData(data, sizeof(data)); // 将包内数据写入到data内
    cout << sock->GetNode()->GetId() << " "
         << "receive : '" << data << "' from " << address.GetIpv4() << endl;
    // 解析data
    char a[sizeof(data)];
    for (uint32_t i = 0; i < sizeof(data); i++)
    {
        a[i] = data[i];
    }
    string strres = string(a);
    stringstream ss(strres);
    vector<string> resultStrings;
    string token;

    while (getline(ss, token, ','))
    {
        resultStrings.push_back(token);
    }
    int ID = sock->GetNode()->GetId();
    int serialnum = stod(resultStrings[4]);
    t1[ID][serialnum] = stod(resultStrings[1]);
    t2[ID][serialnum] = stod(resultStrings[2]);
    t3[ID][serialnum] = stod(resultStrings[3]);
    t4[ID][serialnum] = t1[ID][serialnum] + delay[ID][serialnum] + delay2[ID][serialnum] + waiting[ID][serialnum];
    // cout << "t4 = " << t4 << endl;
    double delta = ((t2[ID][serialnum] - t1[ID][serialnum]) - (t4[ID][serialnum] - t3[ID][serialnum])) / 2;
    deltaTotal[ID] += delta;

    cout << "delta = " << deltaTotal[ID] / (serialnum + 1) << endl;
    cout << "误差：" << abs(TimeShift[ID] + deltaTotal[ID] / (serialnum + 1)) << endl;
    total_error[serialnum] += abs(TimeShift[ID] + deltaTotal[ID] / (serialnum + 1));
    cout << "原始误差：" << abs(TimeShift[ID]) << endl;
    cout << "第 " << serialnum << " 轮" << endl;
    if (serialnum > 0)
    {
        cout << "节点平均误差: " << total_error[serialnum - 1] / numofnodes << endl;
    }
    
    if(serialnum == serialNumMax-1){
        TimeShift[ID] += (deltaTotal[ID] / (serialnum + 1));
        cout << endl;
        cout << "簇头节点"<< ID <<"的同步后误差" << TimeShift[ID] << endl;
    }
    cout << endl;
}

```





