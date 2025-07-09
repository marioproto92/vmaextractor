#!/usr/bin/env python3
import os
import sys
import hashlib
import argparse
import time
import threading
from concurrent.futures import ThreadPoolExecutor

class VmaHeader:
    def __init__(self, fo, skip_hash):
        magic = fo.read(4)
        assert magic == b'VMA\0'
        version = int.from_bytes(fo.read(4), 'big')
        assert version == 1
        self.uuid = fo.read(16)
        self.ctime = int.from_bytes(fo.read(8), 'big')
        self.md5sum = fo.read(16)
        self.blob_buffer_offset = int.from_bytes(fo.read(4), 'big')
        self.blob_buffer_size = int.from_bytes(fo.read(4), 'big')
        self.header_size = int.from_bytes(fo.read(4), 'big')
        fo.seek(1984, os.SEEK_CUR)
        self.config_names = [int.from_bytes(fo.read(4), 'big') for _ in range(256)]
        self.config_data  = [int.from_bytes(fo.read(4), 'big') for _ in range(256)]
        fo.seek(4, os.SEEK_CUR)
        self.dev_info = [VmaDeviceInfoHeader(fo, self) for _ in range(256)]
        fo.seek(1, os.SEEK_CUR)
        self.blob_buffer = {}
        end = self.blob_buffer_offset + self.blob_buffer_size
        while fo.tell() < end:
            offset = fo.tell() - self.blob_buffer_offset
            self.blob_buffer[offset] = Blob(fo)
        fo.seek(self.header_size, os.SEEK_SET)
        self.generated_md5sum = None if skip_hash else self._gen_md5sum(fo)

    def _gen_md5sum(self, fo):
        pos = fo.tell()
        fo.seek(0)
        h = hashlib.md5()
        data = fo.read(self.header_size)
        data = data[:32] + b'\0'*16 + data[48:]
        h.update(data)
        fo.seek(pos)
        return h.digest()

class VmaDeviceInfoHeader:
    def __init__(self, fo, header):
        self.header = header
        self.device_name = int.from_bytes(fo.read(4), 'big')
        fo.seek(4, os.SEEK_CUR)
        self.device_size = int.from_bytes(fo.read(8), 'big')
        fo.seek(16, os.SEEK_CUR)
    def get_name(self):
        name = self.header.blob_buffer[self.device_name].data
        return name.split(b'\0')[0].decode()

class VmaExtentHeader:
    def __init__(self, fo, header, skip_hash):
        self.start = fo.tell()
        magic = fo.read(4)
        assert magic == b'VMAE'
        fo.seek(2, os.SEEK_CUR)
        self.block_count = int.from_bytes(fo.read(2), 'big')
        self.uuid = fo.read(16)
        self.md5sum = fo.read(16)
        self.blockinfo = [Blockinfo(fo) for _ in range(59)]
        self.end = fo.tell()
        self.generated_md5sum = None if skip_hash else self._gen_md5sum(fo)

    def _gen_md5sum(self, fo):
        pos = fo.tell()
        fo.seek(self.start)
        h = hashlib.md5()
        data = fo.read(self.end - self.start)
        data = data[:24] + b'\0'*16 + data[40:]
        h.update(data)
        fo.seek(pos)
        return h.digest()

class Blob:
    def __init__(self, fo):
        self.size = int.from_bytes(fo.read(2), 'little')
        self.data = fo.read(self.size)

class Blockinfo:
    CLUSTER_SIZE = 65536
    def __init__(self, fo):
        self.mask = int.from_bytes(fo.read(2), 'big')
        fo.seek(1, os.SEEK_CUR)
        self.dev_id = int.from_bytes(fo.read(1), 'big')
        self.cluster_num = int.from_bytes(fo.read(4), 'big')


def extract_configs(fo, args, header):
    for i, name_offset in enumerate(header.config_names):
        if not name_offset: continue
        fname = header.blob_buffer[name_offset].data.split(b'\0')[0].decode()
        data = header.blob_buffer[header.config_data[i]].data
        with open(os.path.join(args.destination, fname), 'wb') as f:
            f.write(data)


def extract(fo, args):
    os.makedirs(args.destination, exist_ok=True)
    fo.seek(0, os.SEEK_END)
    total = fo.tell()
    fo.seek(0)

    header = VmaHeader(fo, args.skip_hash)
    if header.generated_md5sum is not None:
        assert header.md5sum == header.generated_md5sum
    extract_configs(fo, args, header)
    fo.seek(header.header_size, os.SEEK_SET)

    # Prepare outputs and locks
    device_outs = {}
    locks = {}
    for idx, dev in enumerate(header.dev_info):
        if dev.device_size > 0:
            path = os.path.join(args.destination, dev.get_name())
            f = open(path, 'wb+')
            device_outs[idx] = f
            locks[idx] = threading.Lock()

    ZERO_CLUSTER = b'\0' * Blockinfo.CLUSTER_SIZE
    cluster_buf = bytearray(Blockinfo.CLUSTER_SIZE)

    def write_cluster(dev_id, pos, data):
        with locks[dev_id]:
            f = device_outs[dev_id]
            f.seek(pos)
            f.write(data)

    executor = ThreadPoolExecutor(max_workers=os.cpu_count() or 4)
    last_print = time.time()
    print('Extracting with multithread...')

    while fo.tell() < total:
        extent = VmaExtentHeader(fo, header, args.skip_hash)
        assert header.uuid == extent.uuid
        if extent.generated_md5sum is not None:
            assert extent.md5sum == extent.generated_md5sum

        for info in extent.blockinfo:
            if info.dev_id == 0: continue
            pos = info.cluster_num * Blockinfo.CLUSTER_SIZE
            mask = info.mask
            # read cluster data
            if mask == 0xFFFF:
                fo.readinto(cluster_buf)
                data = bytes(cluster_buf)
            elif mask == 0:
                data = ZERO_CLUSTER
            else:
                buf = bytearray(Blockinfo.CLUSTER_SIZE)
                for i in range(16):
                    if (mask >> i) & 1:
                        chunk = fo.read(4096)
                        buf[i*4096:(i+1)*4096] = chunk
                data = bytes(buf)
            # submit write task
            executor.submit(write_cluster, info.dev_id, pos, data)

        # progress
        now = time.time()
        if now - last_print >= 1:
            pct = fo.tell() / total * 100
            sys.stdout.write(f"Progress: {pct:.2f}%\r")
            sys.stdout.flush()
            last_print = now

    # wait for tasks to finish
    executor.shutdown(wait=True)
    print('\nClosing files...')
    for f in device_outs.values():
        f.close()
    print('Extraction complete.')

BANNER = """
__     ____  __    _             _                  _             
\ \   / /  \/  |  / \   _____  _| |_ _ __ __ _  ___| |_ ___  _ __ 
 \ \ / /| |\/| | / _ \ / _ \ \/ / __| '__/ _` |/ __| __/ _ \| '__|
  \ V / | |  | |/ ___ \  __/>  <| |_| | | (_| | (__| || (_) | |   
   \_/  |_|  |_/_/   \_\___/_/\_\\__|_|  \__,_|\___|\__\___/|_|   
   Author: Mario Protopapa (Alias DeBuG)
"""
def main():
    print(BANNER)
    parser = argparse.ArgumentParser()
    parser.add_argument('filename')
    parser.add_argument('destination')
    parser.add_argument('-f', '--force', action='store_true', help='overwrite destination')
    parser.add_argument('--skip-hash', action='store_true', help='skip md5 validation')
    args = parser.parse_args()
    if not os.path.exists(args.filename):
        print('Error: source not found')
        return 1
    if os.path.exists(args.destination) and not args.force:
        print('Error: destination exists (use -f)')
        return 1
    with open(args.filename, 'rb') as fo:
        extract(fo, args)
    return 0

if __name__ == '__main__':
    sys.exit(main())
