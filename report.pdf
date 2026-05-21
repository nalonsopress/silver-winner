import selectors
import socket
import sys
import types
import time
import random
import string
import numpy as np

random.seed(10)
#ip address = "192.168.1.51"
sel = selectors.DefaultSelector()
messages_number = 1000000
min_length = 60

mean_del = 0.11 # mean of delay in seconds
std_del = 0.05  # std of delay in seconds
fluctuation_amplitude = 0.3       # Amplitude of fluctuations
breakout_point = 0.001             #threshold for traffic increase
peak_width = 1         # Width of peak

def generate_random_string(length):
    return ''.join(random.choices(string.ascii_letters + string.digits, k=length))

def generate_strings():
    strings = []
    for _ in range(messages_number):
        # Generate a random length based on Gaussian distribution
        length = max(int(random.gauss(100, 20)), min_length)
        random_string = generate_random_string(length)
        strings.append(random_string)
    return strings

messages = generate_strings()

def start_connections(host, port, num_conns):
    server_addr = (host, port)
    for i in range(0, num_conns):
        connid = i + 1
        print(f"Starting connection {connid} to {server_addr}")
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setblocking(False)
        sock.connect_ex(server_addr)
        events = selectors.EVENT_READ | selectors.EVENT_WRITE
        data = types.SimpleNamespace(
            connid=connid,
            msg_total=sum(len(m.encode('utf-8')) for m in messages),
            recv_total=0,
            messages=messages.copy(),
            outb=b"",
        )
        sel.register(sock, events, data=data)


def service_connection(key, mask, timer, last_delay, last_index, growth_time):
    sock = key.fileobj
    data = key.data
    if mask & selectors.EVENT_READ:
        recv_data = sock.recv(1024)  # Should be ready to read
        if recv_data:
            #print(f"Received {recv_data!r} from connection {data.connid}")
            data.recv_total += len(recv_data)
        if not recv_data or data.recv_total == data.msg_total:
            #print(f"Closing connection {data.connid}")
            sel.unregister(sock)
            sock.close()
        delay = 0.1
    if mask & selectors.EVENT_WRITE:
        if not data.outb and data.messages:
            # Normal fluctuation behavior
            if last_index ==  0:
                if last_delay > breakout_point:
                    #k = timer /0.1
                    delay = max(0.001, - abs(np.sin(0.0001 * timer))*fluctuation_amplitude + random.gauss(mean_del, std_del))
                    time.sleep(delay)
                    breakout_ind = 0
                    print(f"inside normal fluctuations: {delay}")
                else:
                    delay = last_delay - random.gauss(mean_del, std_del)
                    delay = max(0.0001,  delay)
                    time.sleep(delay)
                    breakout_ind = 1
                    growth_time = timer
                    print(f"changing to growth: delay = {delay}")
            elif last_index == 1:
                if (timer - growth_time) < peak_width:
                    delay = last_delay - np.power((timer-growth_time),2)
                    delay = max(0.00001,  delay)
                    time.sleep(delay)
                    breakout_ind = 1
                    print(f"inside growth time: timer - growth_time = {timer - growth_time}, delay = {delay}")
                else:
                    delay = last_delay + 0.01
                    breakout_ind = 0
                    growth_time = 0
                    print(f"exiting growth time: delay = {delay}")
            else: delay = 0.1
            data.outb = data.messages.pop(0).encode('utf-8')
        if data.outb:
            #print(f"Sending {data.outb!r} to connection {data.connid}")
            sent = sock.send(data.outb)  # Should be ready to write
            data.outb = data.outb[sent:]
    return (delay, breakout_ind, growth_time)


if len(sys.argv) != 4:
    print(f"Usage: {sys.argv[0]} <host> <port> <num_connections>")
    sys.exit(1)

host, port, num_conns = sys.argv[1:4]
start_connections(host, int(port), int(num_conns))

timer = 0
try:
    while True:
        events = sel.select(timeout=1)
        if events:
            last_delay = 0.1
            last_index = 0
            growth_time2 = 0
            for key, mask in events:
                timer +=0.2
                delay, index, growth_time = service_connection(key, mask, timer, last_delay, last_index, growth_time2)
                last_delay = delay
                last_index = index
                growth_time2 = growth_time
        # Check for a socket being monitored to continue.
        if not sel.get_map():
            break
except KeyboardInterrupt:
    print("Caught keyboard interrupt, exiting")
finally:
    sel.close()
