Stop all images
> docker stop $(docker ps -a -q)

Remove all containers & images ever
> docker system prune -a
