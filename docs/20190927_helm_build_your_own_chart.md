# Create Your Own Helm Chart Repo

> Reference > https://github.com/BurdenBear/kube-charts-mirror

Pre-requisite: you need a server outside of GFW

#### Fork ```https://github.com/BurdenBear/kube-charts-mirror```

#### Clone the Code on Your Server in Free World

```bash
git clone https://github.com/j3ffyang/kube-charts-mirror.git
```

#### Build Docker Image on Server in Free World

```bash
jeff@ubuntu:~$ cd kube-charts-mirror/
jeff@ubuntu:~/kube-charts-mirror$ docker build -t kube-charts-updater .
```

#### Launch the Container

```bash
docker run --name kube-charts \
  -e GIT_REPO="https://j3ffyang:{SECRET}@github.com/j3ffyang/kube-charts-mirror.git" \
  -e GIT_USER_NAME=j3ffyang \
  -e GIT_USER_EMAIL=j3ffyang@gmail.com \
  -v /data/charts:/mnt/charts -d kube-charts-updater
```

The updater will download all chart repo from ```googleapis.com``` or ```gcr.io``` or wherever GFW doesn't like, then upload to your project at GitHub.

#### Your Own Chart Repo

```bash
https://github.com/j3ffyang/kube-charts-mirror/tree/master/docs
```

> Credit and Thanks > https://github.com/BurdenBear/kube-charts-mirror
