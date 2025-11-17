# cfastblf

Cython 加速的 BLF 文件读写和信号查询库。

## 项目简介

`cfastblf` 是一个使用 Cython 编写的高性能 BLF (Binary Logging Format) 文件读写库，专门用于处理汽车总线数据。通过 Cython 将关键性能路径编译为本地代码，提供比 python-can 更快的文件解析和处理速度。

该库支持 CAN/CAN FD 消息的读取、过滤、信号解码和导出，提供底层直接访问和高层查询器两种使用方式，适用于汽车数据分析和测试验证场景。

## 特性

- **高性能解析**: 使用 Cython 编译核心逻辑，显著提升 BLF 文件解析速度
- **NumPy 集成**: 使用 NumPy 结构化数组作为内存数据结构，便于高速数据分析和处理
- **内存高效**: 优化的索引-数据块存储结构，有效减少内存占用，支持一次性加载亿级帧数据
- **完整协议支持**: 支持 CAN、CAN FD、错误帧等多种消息类型
- **灵活的信号解码**: 支持基于 DBC 文件的高速信号提取和转换
- **高级查询功能**: 提供基于表达式的消息过滤和信号映射
- **平台兼容性**: 免费版支持 Windows，其它系统版本请联系作者

## 安装方式

```bash
pip install cfastblf
```

## 使用方法

```python
from cfastblf import read, write, signal_decoder, Query

with open(r'myblf.blf', 'rb') as blf # 注意用二进制读模式
    # 读取 BLF 文件中所有CAN/CANFD帧，返回值为二元组
    # 二元组分别为：索引数组（numpy结构化数组）和紧凑数据块（numpy 字节数组）
    # index 结构化数组的 dtype 结构为：np.dtype([('timestamp', 'i8'), ('can_id', 'u4'), ('is64', 'u1'), ('channel', 'u1'), ('len', 'u1'), ('flags', 'u1'), ('loc', 'u8')])
    # 其中：
    # * timestamp: 时间戳，单位为纳秒
    # * len: 消息负载长度（字节）
    # * flags: 标志位，从高位到低位: reserved, is_extended_id, is_remote_frame, is_error_frame, is_fd, is_rx, bitrate_switch, error_state_indicator
    # * loc: 数据块中的偏移量（字节）
    frames = read(blf) 

# 底层使用方式：直接读取消息帧
index, chunk = frames # 解构二元组，chunk可被帧子集复用
msg = index[5] # 第5个CAN消息（帧）
can_id = msg['can_id'] # 读取 CAN ID
payload = chunk[msg['loc']:msg['loc'] + msg['len']] # 获取消息负载

# 底层使用方式：过滤特定CAN_ID的帧
sub = frames[0][frames[0]['can_id'] == 0x150] # 纯numpy计算

# 高级使用方式：使用查询器
from cantools import database as db
dbc = db.load_file(r'mybus.dbc', encoding='gbk') # 加载 DBC 文件
qry = Query() # 创建查询器
qry.addDatabase(dbc) # 添加信号数据库
# 过滤出 CAN ID 为 0x150 的帧，注意查询字符串中必须用frames代表实际传入的帧数组
sub = qry.filter(frames, "frames['can_id'] == 0x150") # 注意这里 sub 仍为二元组
# 解码帧子集sub中所有帧中 ECU_150_CheckSum 信号
sig = qry.map(sub, 'signals.ECU_150_CheckSum(frames)') # 返回浮点数组

# 将帧子集保存为 BLF 文件
with open('output.blf', 'wb') as f: # 必须二进制写模式
    write(f, sub)
```

## 免费版限制
本模块为免费软件，而非开源软件，免费版限制如下：
1. 仅提供适用于 Windows 系统的二进制安装包。
2. 仅支持处理最多 2M CAN/CANFD 帧，超出此限制的数据将被截断。

## 联系作者

**邮箱:** rockswang@foxmail.com
**项目主页:** https://github.com/rockswang/cfastblf