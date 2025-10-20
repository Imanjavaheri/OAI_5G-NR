# OAI 5G Softmodem Simulation & Data Test (RFsim + noS1)

This guide explains how to run the OpenAirInterface (OAI) 5G gNB and UE
softmodems using the RF simulator and how to validate user plane data flow via
`iperf` while operating in `--phy-test --noS1` mode. The procedure requires no
dedicated RF hardware and bypasses core network connectivity.

## Prerequisites

1. Clone the OAI source repository (for example, into `~/openairinterface5g`).
2. Install the required tools:
   ```bash
   sudo apt install iperf netcat
   ```
3. Prepare four separate terminal windows so the softmodems and traffic tools
   can run concurrently.

## Phase 1 – Build Softmodems (if not already built)

These steps ensure the `nr-softmodem` and `nr-uesoftmodem` binaries are built
for the simulator configuration.

```bash

# 1. Change to the root OAI directory:
cd ~/openairinterface5g

# 2. Clean previous build artifacts (recommended):
./cmake_targets/build_oai --clean

# 3. Build the gNB and NR UE executables using simulated hardware:
./cmake_targets/build_oai --gNB --nrUE -w None
```

## Phase 2 – Start Softmodems (Connection Setup)

Open two terminals—one for the gNB and another for the UE. Leave both running
in the foreground for the duration of the test.

### Terminal 1 (gNB – Server)

The gNB creates the virtual interface `oaitun_enb1` with IP `10.0.1.1`.

```bash
# Change to the build directory:
cd ~/openairinterface5g/cmake_targets/ran_build/build

# Run gNB with RFsim, PHY-Test, noS1, and HARQ timing adjustments:
sudo ./nr-softmodem -O \
  ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
  --gNBs.[0].min_rxtxtime 6 --rfsim --phy-test --noS1
```

### Terminal 2 (UE – Client)

The UE creates the virtual interface `oaitun_ue1` with IP `10.0.1.2`.

```bash
# Open a new terminal and change to the build directory:
cd ~/openairinterface5g/cmake_targets/ran_build/build

# Run UE with matching radio parameters and testing flags:
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 \
  --ssb 516 --rfsim --phy-test --noS1
```

**Wait:** Allow the UE to synchronize and stabilize the connection before
continuing.

## Phase 3 – Automated Data Test (iperf)

Use two additional terminals to verify downlink user plane data flow.

### Terminal 4 (iperf Server – UE Side)

Listen for incoming data on the UE’s IP address.

```bash
iperf -sui1 -B 10.0.1.2
```

### Terminal 3 (iperf Client – gNB Side)

Send a 1 Mbps UDP stream from the gNB IP to the UE IP for 10 seconds.

```bash
iperf -uc 10.0.1.2 -B 10.0.1.1 -i1 -t10 -b1M
```

**Result:** Throughput statistics appear in both terminals, confirming user
plane traffic across the simulated 5G link.

## Phase 4 – Manual Message Sending (Ping & Netcat)

After `iperf`, you can run simple connectivity tests.

### 1. Connectivity Check (ping)

From Terminal 3, send ICMP echo requests from the gNB to the UE.

```bash
ping -c 4 10.0.1.2
```

You should receive successful replies for each packet.

### 2. Custom Text Message (netcat)

Use `netcat` to send a message across the simulated link.

#### Terminal 4 – UE Listener

Stop the `iperf` server (Ctrl + C), then start the listener:

```bash
nc -l 10.0.1.2 5555
```

#### Terminal 3 – gNB Sender

Connect to the UE listener and send your message:

```bash
nc 10.0.1.2 5555
```

Type your message (for example, `Hello OAI!`), press **Enter**, then press
**Ctrl + D** to close the connection.

**Success:** The message appears immediately in Terminal 4.
