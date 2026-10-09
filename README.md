# Arista_Notes

**All this is done in my home lab**

Go into "my-arista-lab" folder and check if the containers are already running
```
docker ps
```

*If the containers are not running*
```
sudo systemctl start docker # launches the background Docker daemon (engine)
sudo systemctl status docker --no-pager # checks the status

sudo containerlab deploy -t {file, ex: topology.clab.yml}

docker start clab-arista-lab-xxx # this starts those containers themselves since they have already been creat ed before


# extra commands when troubleshooting:
docker ps -a # shows all containers even if the container is not running
```

Starting the CLi of the Router/Switch
```
docker exec -it clab-mlag-lab-xxx Cli
```