# cfastblf

Cython-accelerated BLF file reading/writing and signal query library.

## Project Introduction

`cfastblf` is a high-performance BLF (Binary Logging Format) file reading/writing library written in Cython, specifically designed for processing automotive bus data. By compiling critical performance paths into native code using Cython, it provides faster file parsing and processing speed compared to python-can.

This library supports reading, filtering, signal decoding, and exporting of CAN/CAN FD messages, offering both low-level direct access and high-level query usage patterns, suitable for automotive data analysis and test validation scenarios.

## Features

- **High-performance parsing**: Core logic compiled with Cython, significantly improving BLF file parsing speed
- **NumPy integration**: Uses NumPy structured arrays as memory data structures for high-speed data analysis and processing
- **Memory efficient**: Optimized index-data block storage structure effectively reduces memory usage, supports loading hundreds of millions of frame data at once
- **Complete protocol support**: Supports multiple message types including CAN, CAN FD, error frames, etc.
- **Flexible signal decoding**: Supports high-speed signal extraction and conversion based on DBC files
- **Advanced query functionality**: Provides expression-based message filtering and signal mapping
- **Platform compatibility**: Free version supports Windows, contact author for other system versions

## Installation

```bash
pip install cfastblf
```

## Usage

```python
from cfastblf import read, write, signal_decoder, Query

with open(r'myblf.blf', 'rb') as blf:  # Note: use binary read mode
    # Read all CAN/CANFD frames from BLF file, returns a tuple
    # The tuple contains: index array (numpy structured array) and compact data block (numpy byte array)
    # The dtype structure of index structured array: np.dtype([('timestamp', 'i8'), ('can_id', 'u4'), ('is64', 'u1'), ('channel', 'u1'), ('len', 'u1'), ('flags', 'u1'), ('loc', 'u8')])
    # Where:
    # * timestamp: timestamp in nanoseconds
    # * len: message payload length (bytes)
    # * flags: flag bits, from high to low: reserved, is_extended_id, is_remote_frame, is_error_frame, is_fd, is_rx, bitrate_switch, error_state_indicator
    # * loc: offset in data block (bytes)
    frames = read(blf)

# Low-level usage: directly read message frames
index, chunk = frames  # Destructure tuple, chunk can be reused by frame subsets
msg = index[5]  # The 5th CAN message (frame)
can_id = msg['can_id']  # Read CAN ID
payload = chunk[msg['loc']:msg['loc'] + msg['len']]  # Get message payload

# Low-level usage: filter frames with specific CAN_ID
sub = frames[0][frames[0]['can_id'] == 0x150]  # Pure numpy computation

# High-level usage: using query
from cantools import database as db
dbc = db.load_file(r'mybus.dbc', encoding='gbk')  # Load DBC file
qry = Query()  # Create query
qry.addDatabase(dbc)  # Add signal database
# Filter frames with CAN ID 0x150, note: must use 'frames' in query string to represent actual frame array
sub = qry.filter(frames, "frames['can_id'] == 0x150")  # Note: sub is still a tuple
# Decode ECU_150_CheckSum signal from all frames in subset sub
sig = qry.map(sub, 'signals.ECU_150_CheckSum(frames)')  # Returns float array

# Save frame subset as BLF file
with open('output.blf', 'wb') as f:  # Must use binary write mode
    write(f, sub)
```

## Free Version Limitations

This module is freeware, not open source software. Free version has the following limitations:
1. Only provides binary installation packages for Windows systems.
2. Only supports processing up to 2M CAN/CANFD frames, data exceeding this limit will be truncated.

## Contact Author

**Email:** rockswang@foxmail.com
**GitHub:** https://github.com/rockswang/cfastblf