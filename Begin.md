# Mấy cái cơ bản ban đầu
## Lý thuyết
- Docker sẽ đóng gói toàn bộ môi trường máy tính vào 1 cái gọi lại image và dễ dàng đem nó chạy trên các máy tính khác.
- Image khi được chạy lên sẽ khởi tạo 1 instance của image và nó được gọi là container
- Vậy đại khái là 1 giải pháp để chạy máy ảo trong một máy khác  tuy nhiên nó nhẹ hơn VM nhiều vì nó chỉ đống gói 1 phần (hơi chung chung) và dùng chung rất nhiều tài nguyên. Bình thường mỗi máy ảo sẽ có 1 kernel riêng, thư viện riêng nhưng Container thì #, nó sài chung luôn ==> giảm được tài nguyên. Hoặc suy nghĩ là VM là ảo hóa mức OS, Docker là ảo hóa mức APP ==> không nên chạy nhiều App trong 1 container
 ![img](images/dockervsvm.png)
 - Docker compose: Dùng để khai báo và run multi-container cho docker. Sử dụng file yaml để config
 ### Dockerfile
- Chứa tập hợp các lệnh để docker đọc và thực thi nhằm mục đích đóng gói 1 image theo yêu cầu của người phát triển. Bằng cách sử dụng *docker build*. Format file là
```
#Comment
Instruction Arguments
```
- FROM: Là base image để chúng ta tiến hành build 1 image mới trên image base. Ví dụ FROM ubuntu
- MAINTAINER: Thông tin người phát triển
- RUN: Sử dụng để thực thi các command trong quá trình build image
- COPY: Copy một file từ host machine tới docker container
- WORKDIR: image sẽ trỏ tới directory khai báo, nếu không có sẽ tạo mới
- ENV: Định nghĩa biến môi trường
- CMD: Sử dụng để thực thi các command trong quá trình build container mới từ image
## Cài đặt
- Bao gồm cả docker client
```
yum update
curl -fsSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
- Thấy network có thêm 1 card docker0
```
18: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:e9:17:00:a3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
- Tiến trình có thêm
```
root  9216  1 TS 19 14:27 ?  00:00:01 /usr/bin/containerd
root  9217  1 TS 19 14:27 ?  00:00:00 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```
## Demo tạo một image chạy nginx trên ubuntu/centos
- Tạo Dockerfile
```
vim Dockerfile
FROM centos:7
CMD echo "Create Image Centos 7
RUN yum -y install epel-release
RUN yum -y update && yum clean all
RUN yum -y vim wget net-tools git
RUN yum -y install nginx
ADD index.html /var/www/html
EXPOSE 80 
CMD [ "nginx", "-g", "daemon off;" ] # Vì docker có kiểm tra tiến trình, nếu ko có thì nó stop container. Nên phải cho Nginx chạy foreground để Docker nhìn thấy process. Và khi run image mà không truyền tham số gì vào thì nó sẽ chạy cái câu lệnh này. Điều này tương đương trong container là "nginx -g 'daemon off;'"

vim Dockerfile
FROM ubuntu:16.04
CMD echo "Hello ubuntu"
#Update the repository
RUN apt-get update
#Install necessary tools
RUN apt-get install -y nano wget dialog net-tools vim git
#Download and Install nginx
RUN apt-get install -y nginx
# Remove file index docker. Add file index.html in host machine to /var/www/html in image
RUN rm -v /var/www/html/index.nginx-debian.html
ADD index.html /var/www/html
EXPOSE 80
CMD [ "nginx", "-g", "daemon off;" ]
```
- Tạo file index.html với nội dung bất kì
- Thực hiện build image
```
docker build -t "mycentos" .
mycentos: Tên image
.: Tức là file Dockerfile đang ở thư mục hiện tại
```
- Thực hiện lauch container
```
docker run -d -p 8080:80 mycentos
Thực hiện chạy image tên mycentos và bind port 8080 local với port 80 của container
Client sẽ truy cập nginx qua IP_Server:8080
```
## Câu lệnh
### Image
- docker images: Hiện thị các image
- docker rmi [name_image]: Xóa Image
- docker rmi -f [name_image]: Xóa image kể cả khi nó đang liên kết tới 1 container
### Docker run ... container
- Tham khảo các parameter trong câu lệnh https://docs.docker.com/engine/reference/commandline/run/
- docker run -p 8080:80 -it mycentos /bin/bash: Thực hiện run image và truy cập vào container (như ssh vô). 
- docker run -d -p 8080:80 mycentos :Thực hiện run image và chạy container ở chế độ ngầm, đồng thời hiện ra container ID
- docker run --rm -d -p 8999:80 mycentos: Thêm --rm nếu muốn khi container mà stop thì xóa đi luôn
- Process của một container running khi show ở host machine
```
/usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8899 -container-ip 172.17.0.5 -container-port 80
containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/59a9ff66ced82e3294bbf92a1bddc579a643c
```
- docker stop [id_container]: Stop container
- docker rm [id_container]" Xóa container
- docker rm $(docker ps -a -f status=exited -q): Xóa toàn bộ container có trạng thái là exited. -f là filter
### Docker build image
### Docker show container
- docker ps: Hiện thị các container đang chạy
- docker ps --all/-a: Hiện thị toàn các các container
- docker ps -a -f status=exited: Hiện thị các container có trạng thái exited

