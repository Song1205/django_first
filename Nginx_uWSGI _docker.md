Install and connect Nginx and uWSGI  
use mysql with docker

pip install uwsgi


worker 연산하는 역할 프로세스 설정하면 여러개 가능(동시에처리가능)

client(웹브라우저)url요청->uwsgi->django
                        <-     <-응답
                        
                       
uwsgi --http :8000 --module server_dev.wsgi
지금은 설정을 명령어로 주지만 커스텀설정파일로 관리하면 편하다

루트경로에서
/etc/uwsgi/sites/ 폴더생성

[uwsgi] base = /home/ubuntu/server_dev 
home = %(base)/venv chdir = %(base) #가상환경 경로
module = server_dev.wsgi:application #프로젝트 경로

socket = /tmp/django.sock #리눅스에서 제공하는 소켓사용 
chmod-socket = 666 #소켓의 파일권한 
master = true #마스터 프로세스 유무 
enable-threads = true # 워커를 쓰레드 기반으로 할 것인가
pidfile = /tmp/django.pid #uwsgi 모든 프로세스 번호를 쉽게 확인
vacuum = true #uwsgi 흔적(soket, pid)을 자동으로 지워줌
logger = file:/tmp/uwsgi.log # log파일 기록


소켓을 리눅스 사용하는 이유=> 네트워크를 쓰면 헤더가 하나 더 붙는다 osi7계층에서 네트워크 계층이 있기 때문에 헤더가 하나 더 붙는다 헤더가 하나 더 붙기 때문에 다이렉트로 통신하는 것보다 속도가 더 떨어진다
마스터 프로세스는 워커프로세스를 감시 부모 프로세스로 생각하면 된다 워커들을 관제하는 마스터프로세스 마스터 로그가 찍히기 때문에 시스템을 더 잘 볼수 있다
쓰레기 사용여부에 따라 차이가 있기 때문에 사용하는 것이 좋다

tail -f uwsgi.log
로그추적

/etc/nginx/nginx.conf
서버개발자에게 config 파일을 각각 어떤 의미인지 아는 게 중요 (면접에서 묻기도 좋음)
sendfile : 응답을 보낼 때 파일을 read(), write()하게 되는데, 사용자 영역에서 파일을 읽고 쓰는 것이 아니라 커널 영역의 buffer를 사용해 속도를 향상하는 옵션입니다.

 

tcp_nopush : sendfile이 on일 때만 사용 가능한 옵션으로, 리눅스에서 TCP_CORK옵션을 사용해 통신합니다. TCP_CORK란 쉽게 설명하면 네트워크 통신에서 패킷을 여러 개를 보내지 않고 한번에 보내서 통신 속도를 향상하는 방법입니다. 왜 여러 번 보낼때보다 한번이 좋냐면, 패킷을 보낼때 항상 헤더 패킷이 앞에 붙기 때문에, 여러번 보내면 전송되는 헤더 패킷이 많아지기 때문입니다.

 

tcp_nodelay : tcp_nodelay는 tcp_nopush와 반대되는 개념입니다. TCP_CORK를 사용하면 패킷을 여러 개 모아서 보낸다고 했는데, tcp_nodelay는 아무리 작은 패킷도 바로바로 전송하는 옵션입니다. 구글링 해보면 1바이트를 보낼 때 헤더 붙고 뭐 붙고 하고, 다른 패킷 없나 찾아보다가 0.2초 정도 늦게 나간다고 합니다. 근데 tcp_nodelay 키면 바로 나가는 거죠. 이 옵션은 keepalive_timeout 옵션이 켜있어야 합니다.



nginx config는 {}를 사용해 "블록"단위로 나누어집니다. 블록밖에 있는 config도 있는데 이를 core 모듈이라고 합니다.

 

1. core 모듈 
 

user : nginx 프로세스가 실행되는 계정을 말합니다. 루트 계정으로 하지는 않고 보통 사용자 계정 중에 하나를 선택합니다.

 

worker_process : nginx 프로세스가 실행되는 수입니다. auto로 하면 서버에 cpu 수만큼 뜹니다.

 

pid : uwsgi 설정에서도 했었는데 nginx 프로세스들의 pid를 모아놓는 파일을 생성하는 설정입니다.

 

include : 이것 어디서나 사용될 수 있는데, 특정 경로에 있는 .conf 파일을 읽어옵니다. 설정 파일들을 나누어 관리할 수 있게 해주는 역할입니다.

 

2. event 블록
 

worker_connections : 하나의 worker 프로세스가 처리할 수 있는 커넥션 수입니다. nginx는 비동기로 처리하기 때문에 하나의 프로세스가 다수의 커넥션을 가질 수 있습니다.

 

multi_accept : nginx가 동시에 커넥션을 받을지를 정하는 config입니다. 위에 worker_connections 설명할 때 하나의 worker 프로세스가 여러 개 커넥션을 갖는다고 했는데 이건 뭐지?라고 생각할 수 있는데, multi_accept는 '동시에 커넥션을 받는가?'에 대한 설정입니다. 기본값은 off인데, off일 경우 한 번에 하나의 커넥션을 생성해서 worker_connections에 설정된 수만큼 생성하는 것이고, multi_accept가 on일 경우 한 번에 50개 혹은 100개씩 커넥션을 생성할 수 있습니다. 이렇게 되면 한번에 들어오는 connection이 worker_connections보다 많으면 에러가 발생하게 되니 상황을 고려해서 on 해야 합니다.

 

3. http 블록
 

sendfile : 응답을 보낼 때 파일을 read(), write()하게 되는데, 사용자 영역에서 파일을 읽고 쓰는 것이 아니라 커널 영역의 buffer를 사용해 속도를 향상하는 옵션입니다.

 

tcp_nopush : sendfile이 on일 때만 사용 가능한 옵션으로, 리눅스에서 TCP_CORK옵션을 사용해 통신합니다. TCP_CORK란 쉽게 설명하면 네트워크 통신에서 패킷을 여러 개를 보내지 않고 한번에 보내서 통신 속도를 향상하는 방법입니다. 왜 여러 번 보낼때보다 한번이 좋냐면, 패킷을 보낼때 항상 헤더 패킷이 앞에 붙기 때문에, 여러번 보내면 전송되는 헤더 패킷이 많아지기 때문입니다.

 

tcp_nodelay : tcp_nodelay는 tcp_nopush와 반대되는 개념입니다. TCP_CORK를 사용하면 패킷을 여러 개 모아서 보낸다고 했는데, tcp_nodelay는 아무리 작은 패킷도 바로바로 전송하는 옵션입니다. 구글링 해보면 1바이트를 보낼 때 헤더 붙고 뭐 붙고 하고, 다른 패킷 없나 찾아보다가 0.2초 정도 늦게 나간다고 합니다. 근데 tcp_nodelay 키면 바로 나가는 거죠. 이 옵션은 keepalive_timeout 옵션이 켜있어야 합니다.

 

keepalive_timeout : 응답을 보내고 얼마나 커넥션을 유지할지를 설정하는 옵션입니다. 여러 번 통신을 하는 거면 커넥션을 맺는 일조차 낭비이기 때문에 이 옵션을 켜는 게 좋고, 단일로 발생한다면 끄는 게 좋습니다. 켜져 있으면 그만큼 커넥션이 살아있는 거라서 리소스를 잡아먹습니다.

 

type_hash_max_size : 호스트 도메인 이름 공간이라고 합니다. 넉넉하게 잡아야 긴 이름 할 수 있습니다.

 

server_tokens : 에러 페이지에 서버 버전 노출할지 옵션. 재밌는 옵션이네

 

server_names_hash_bucket_size : 서버 이름을 저장하고 있는 해쉬인데 도메인이 늘어나면 늘려줘야 한다고 하네요

 

server_name_in_redirect : on일 경우 redirect로 요청이 올 경우 server_name에 첫 번째 이름을 사용합니다.

 

include /etc/nginx/mime.types, default_type : 이 두 개는 쌍으로 붙어있는데, mime.type을 열어보면 파일 확장자 별로 어떤 type의 데이터인지 정의되어있습니다. 정의되어있지 않는 type일 경우 default_type에 정의된 type으로 인식합니다. application/octet-stream으로 되어있는데 바이너리 파일로 인식한다는 뜻입니다.

 

ssl_protocols : ssl 프로토콜을 어떤 것을 사용할지 명시되어있습니다. 거의 건 드는 일이 없다고 합니다.

 

ssl_prefer_server_ciphers : sslv3이나 TLS 프로토콜을 사용할 때 서버에 있는 암호화 방법을 사용할지 정하는 옵션입니다. 

 

access_log, error_log : 로그파일의 경로를 지정하는 옵션입니다.



출처:https://cholol.tistory.com/485?category=966420
