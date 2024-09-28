# This directory will be used to follow the lessons and store the files related to the Oreilly Course: [Red Hat Certified Specialist in Containers (EX188)](https://learning.oreilly.com/course/red-hat-certified/9780135335956/)


## Firs step is to create your environment using DevBox

Check the DevBox Readme `../devbox.md`

## Lesson 1: Why use Containers

This lesson covers some concepts about containers and how Podman beats Docker (newer) and still provides compatibilty with it utilizing the OCI standatds.


## Lesson 2: Working with podman

### 2.1 Registry Access

```bash
## This needs developer redhat account (it is free)
podman login registry.redhat.io  

# Before that I had to create some confguration files to make it work on Ubuntu (not necessary if you are using CentOs or Rhel)

podman search ubi

podman pull registry.redhat.io/ubi9/ubi
```

### 2.2 Running Containers