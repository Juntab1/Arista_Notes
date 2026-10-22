# Arista Notes

This repo is a collection of notes from my home lab, mainly focused on Arista switching and Containerlab-based lab builds.

## Lab startup

From the lab folder, check whether the containers are already up:

```bash
docker ps
```

If they are not running, start Docker and redeploy the lab:

```bash
sudo systemctl start docker
sudo systemctl status docker --no-pager
sudo containerlab deploy -t topology.clab.yml
```

If the containers already exist but are stopped, you can bring them back up with:

```bash
docker ps -a
docker start clab-arista-lab-xxx
```

Troubleshooting tip:

```bash
docker ps -a
```

This shows all containers, including ones that are not currently running.

## Open the CLI for a switch or router

```bash
docker exec -it clab-mlag-lab-xxx Cli
```

Replace `xxx` with the node name you want to access.

## Adding a server to the lab

If you add a server or another node to the lab, you need to update the Containerlab topology file and redeploy the lab:

```bash
sudo containerlab deploy -t topology.clab.yml
```

The typical workflow is:

1. Add the new node to the YAML topology
2. Define the interface or network connection
3. Redeploy Containerlab
4. Bring the device up and configure the interface
5. Verify reachability from the switch or leaf

Example:

```bash
docker exec -it clab-mlag-lab-server1 sh
ip addr
```

Then configure the server interface and test connectivity:

```bash
ip addr add 10.0.10.10/24 dev eth1
ip link set eth1 up
ip route add default via 10.0.10.1
ping 10.0.10.1
```

This is the same basic pattern used elsewhere in the lab: add the device, connect it to the right VLAN or routed interface, then verify it can reach the network.

## Notes

For the detailed MLAG and spine setup, see [MLAG.md](C:/Users/jbruttomesso/Arista_Notes/MLAG.md).