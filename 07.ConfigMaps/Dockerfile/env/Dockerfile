FROM ubuntu:16.04
# Ali apt-get source.list
RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial main" >/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial main" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial universe" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security main" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe" >>/etc/apt/sources.list 

RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop-env.sh /bin/fortuneloop.sh
RUN chmod +x /bin/fortuneloop.sh
# RUN mkdir /var/htdocs

ENTRYPOINT ["/bin/fortuneloop.sh"]