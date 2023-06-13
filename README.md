# AquaNet experiments instructions

This instruction describes how to run `ALOHA` and `TRUMAC` protocols in AquaNet on Raspberry Pis.

## Experimental setup

We assume that each Raspberry Pi has an acoustic modem attached, accessible over `/dev/ttyUSB0` serial interface. In addition, we assume the following:

- 4 Raspberry Pi devices
- 4 acoustic modems
- all modems can communicate with each other

All Raspberry Pis have `AquaNet` software compiled and pre-installed. The folder is located under the following path:
`/home/pi/aquanet/`

## AquaNet configuration

All configuration files are located under the following path:

`/home/pi/aquanet/test_trumac/nodeX/`

where `X` - the host ID of Raspberry Pi:

- RPi1 (100.74.94.63): X = 1
- RPi2 (100.88.44.95): X = 2
- RPi3 (100.120.115.117): X = 3
- RPi4 (100.73.149.116): X = 4

There are multiple files under the path, the most important files are:

- `run_poisson.sh`:

This is the main `aquanet` script that starts AquaNet stack and runs the experiment in foreground. The contents of the script are the following:

```
# start the protocol stack
../../bin/aquanet-stack &
sleep 2
# start the VMDM
../../bin/aquanet-gatech &
sleep 4
# start the MAC
# ../../bin/aquanet-trumac &
../../bin/aquanet-uwaloha &
sleep 2
# start the routing protocol
../../bin/aquanet-sroute &
sleep 2
# start the transport layer
../../bin/aquanet-tra &
sleep 2
# start the application layer
../../bin/aquanet-periodic-sender 1 2 50000 0.1 &
```

This script defines application parameters and the MAC protocol that is currently in use. To change the application parameters, the last line in the script should be modified:

`../../bin/aquanet-periodic-sender 1 2 50000 0.1 &`

The line above runs a `periodic-sender` application with the **source address** equals to `1`, **destination address** equals to `2`, **traffic period** equals to `50000` milliseconds, and the **period jitter** equals to 10%.

To switch between `TRUMAC` and `ALOHA` protocols, the corresponding protocol should be enabled by commenting out the other protocol's line. I.e., the lines below enable `TRUMAC` protocol and disable `ALOHA`:

```
../../bin/aquanet-trumac &
#../../bin/aquanet-uwaloha &
```

- `stack-stop.sh`:

This is a utility script that stops all the process spawned by `run_poisson.sh` script.

- `config_ser.cfg`:

This is a configuration file of the modem's serial interface. First line defines a path to the serial interface (`/dev/ttyUSB0` by default), the second line defines the baud rate (`115200` by default).

- `aquanet-trumac.cfg`:

This is a configuration file of `TRUMAC` protocol. First column defines a maximum number of nodes in the experiment (`4` by default). Second column defines a contention timeout (`300000ms` by default). Third column defines a guard interval between two consequent transmissions (`1000ms` by default).

**Note:** `ALOHA` protocol has no separate configuration file.

## Gathering performance statistics

In the experiments, we are going to compare performance of `ALOHA` and `TRUMAC` protocols. For that, each RPi will run the main `aquanet` process that will generate periodic packet traffic and pass it over to MAC layer. The MAC layer will process packets from the application layer and send the final frames to the modem over serial interface.

To assess MAC performance, `periodic-sender` application will log all transmission (TX) and reception (RX) events that will happen at the application layer, and store them in `app_trace.tr` file. The format of the file is the following:

```
>>> cat app_trace.tr 
t
t
t
r
t
r
t
r
t
```

where `t` - corresponds to the TX event; `r` - correspond to the RX event at the application layer.

Based on total number of TX and RX events counted in `app_trace.tr` file, we can calculate **Packet Delivery Ratio (PDR)**, which is going to be our main performance metric:

**PDR = N_RX / N_TX**

where: **N_RX** - total number of RX events across all the nodes; **N_TX** - total number of TX events across all the nodes.

To count the number of TX/RX events, a combination of `grep` and `wc` commands can be used. For example, the following command counts the number of RX events in the experiment at a particular node:

`grep "r" app_trace.tr | wc`


## Example: running an experiment

In this example we are going to run `AquaNet` process with `periodic-sender` application with traffic period of **100 seconds** and `ALOHA` protocol running on 4 nodes (RPis).

### Step 1: Configure `run_poisson.sh` script at each node

Each node should have `run_poisson.sh` configured under `/home/pi/aquanet/test_trumac/nodeX/` path, containing the following lines:

```
# start the protocol stack
../../bin/aquanet-stack &
sleep 2
# start the VMDM
../../bin/aquanet-gatech &
sleep 4
# start the MAC
# ../../bin/aquanet-trumac &
../../bin/aquanet-uwaloha &
sleep 2
# start the routing protocol
../../bin/aquanet-sroute &
sleep 2
# start the transport layer
../../bin/aquanet-tra &
sleep 2
# start the application layer
../../bin/aquanet-periodic-sender X Y T 0.1 &
```

where: **X** - address of the source node; **Y** - address of the destination node; **T** - traffic period in milliseconds.

**Note:** the node addresses match the node/host IDs, i.e. RPi1 has node address of **1**, RPi2 has node address of **2**, etc.

In the configuration, **X** must be equal to the **node ID** where the application script is running. **Y** can be any other node ID. E.g., X=1 and Y=2 at node 1; X=2 and Y=1 at node2; X=3 and Y=4 at node 3; X=4 and Y=3 at node 4, etc.

After configuration, the script should be started on all the 4 nodes by executing:

```
>>> ./run_poisson.sh
```

Alternatively, the process can also be started in the background by running:

```
>>> nohup ./run_poisson.sh &
```

While the script is running, the `app_trace.tr` file will get periodically updated. The content of the file can be checked either by `cat` or `tail` command, e.g.:

```
>>> tail -f app_trace.tr
```

When the sufficient amount of `t` and `r` events are gathered in `app_trace.tr` file, the experiment can be stopped by executing `./stack-stop.sh`. Now, the `PDR` metric can be calculated using the formula above.

**Note**: To calculate `PDR`, **N_TX** should be the sum of all `t` events across all 4 nodes; and **N_RX** should be the sum of all `r` events across all 4 nodes.

## Troubleshooting

This section will get updated as issues accumulate.

### Modem connection check

Before running any experiments, **it is very important** to check physical connectivity / channel communication among the modems. Otherwise, the number of RX events may be close to zero, since the underwater channel becomes too unreliable.

To check the connectivity, `echo` and `minicom` commands can be used. For example, at `node1` we can send an arbitrary `hello` string by executing:

```
>>> echo 'hello' >> /dev/ttyUSB0
```

At the receivers (nodes 2, 3 and 4), we can start `minicom` program, that should receives the `hello` string from `node1` and show it in the output:

```
Welcome to minicom 2.8

OPTIONS: I18n 
Port /dev/ttyUSB0

Press CTRL-A Z for help on special keys

hello
```

If the message is not appearing, please check the modem's connectivity before running AquaNet experiments.
