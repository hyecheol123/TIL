# 2020-08-24 Let's Encrypt 와일드카드 TLS(SSL) 인증서 with Cloudflare + 아파치 설정
최초 작성일: Aug. 24. 2020


#### Table of Contents
- Introduction
- Let's Encrypt: Get WildCard Certificate with Cloudflare DNS
- Apache Virtual Host Setup
- Conclusion


### Introduction
웹 브라우저에서 주소창에 자물쇠 표시가 된 것을 본적이 있을 것이다.
이는 TLS를 활용한 암호화된 HTTPS(HTTP Secure) 프로토콜을 활용하는 웹사이트에 한해 웹 브라우저에서 아이콘을 표시해 주는 것으로써, HTTPS 프로토콜을 사용하는 웹사이트의 경우 HTTP 프로토콜을 사용하는 웹사이트보다 중간자공격 및 도청의 위험에서 **조금 더 안전**하다는 장점이 있다.

특히, 사이트 이용자의 개인정보를 이용해야만 하는 경우, 법적으로 보안연결이 강제되는 경우도 있을 뿐더러, 최신 버전의 웹 브라우저에서 HTTPS 미사용 사이트의 경우 일부 기능을 제한하는 방법으로 HTTPS 사용을 강제하기도 한다.  

이름있는 **신뢰할 수 있는 인증기관**에서 인증서를 발급받을려면 ~~(꽤 큰)~~ 비용이 들며, 개인 사이트에 단순하게 HTTPS를 지원하기 위해 받는 TLS(SSL) 인증서에 돈을 들이기에는 아깝다고 생각되는 것이 사실이다.
공신력있는 대형 인증기관(CA)의 인증서 사용을 통한 서비스의 신뢰도 향상이나 인증서 자체의 취약에 따른 보상 혜택은 없지만, 충분히 신뢰할 수 있고, 보안성 자체만으론 큰 차이가 없는 **무료 TSL(SSL) 인증서**를 발급해주는 **Let's Encrypt**를 활용해 인증서를 발급받는 방법을 알아보자.  


### Let's Encrypt: Get WildCard Certificate with Cloudflare DNS
**와일드카드 인증서**란, 특정 서브도메인 에서만 사용할 수 있는 인증서가 아닌, `*.example.com` 형식으로 발급되어 **모든 서브도메인에서 동시에 사용할 수 있는 인증서**를 의미한다.
Let's Encrypt로 와일드카드 인증서 또한 발급받을 수 있다.
이를 위해서는 도메인 DNS에 TXT 레코드를 생성하여 도메인의 소유권을 증명하여야 하며, 갱신시마다 TXT 레코드를 새로이 생성해 주어야 하므로, DNS가 API를 제공하지 않는 경우 갱신 과정에서 일일히 TXT 레코드를 등록해주어야 한다.  

Cloudflare에 대해 간략히 설명하면, 도메인 네임서버 설정 하나만으로 DNS, CDN, 디도스 방어 등의 서비스를 무료로 서비스하는 업체로, 트래픽 절감 및 서버의 IP 노출 방지 등의 장점으로 인해 꾸준히 사용해왔다.
Cloudflare는 모든 기능을 API로 제어할 수 있도록 되어있으며, 이는 반복되는 DNS 레코드 변경을 자동화시킬 수 있도록 해준다.
필자는 이를 이용해 유동IP 환경에서도 서버를 운영할 수 있도록 자동으로 바뀐 IP주소를 Cloudflare에 업데이트해주는 [스크립트](https://github.com/hyecheol123/Cloudflare-Public-Dynamic-IP-Update)를 만들었었다.
이번에는 **Cloudflare의 DNS를 활용하는 도메인의 와일드카드 인증서를 Let's Encrypt를 이용해 발급받는 방법을 알아보도록 하자.**  

***아래에서 쓸 모든 명령은 root 계정에서 수행되어야 한다.**  
**작업 환경**: Ubuntu 20.04 LTS + Apache (다른 데비안 기반 리눅스 배포판에서도 같은 명령어로 작업 가능)*  

#### Cloudflare API 정보 입력
인증서 신규발급 및 갱신시마다 Cloudflare DNS의 레코드를 변경해주어야 한다.
클라우드플레어에서 제공하는 API를 통해 이를 자동화 할 수 있으며, 이를 위해선 **API 토큰**을 필요로 한다.   
DNS 레코드의 변경을 위해서는 `Zone:DNS:Edit` 권한을 필요로 한다.
발급방법에 대해서는 [Cloudflare의 공식 가이드](https://support.cloudflare.com/hc/en-us/articles/200167836-Managing-API-Tokens-and-Keys)를 참고하면 된다.  

*Global API Key를 사용할 수도 있지만, 이는 계정의 모든 권한을 쥐고 있는 API Key로 이를 직접 사용하는 것은 부적절하다.
제한된 권한만을 가지고 있는 API Token을 사용하자.*  

API 토큰 등 인증정보는 서버의 관리자만이 접근하여야 하고, 다른 사람은 접근할 수 없어야 한다.
root(`/root`)의 홈 디렉터리 안에 각종 인증정보를 저장할 폴더(`.secret`)를 만들고, Cloudflare의 인증 정보를 담은 파일(``)을 만든다.
여기서는 `vim` 편집기를 사용하였지만, 기호에 따라 `nano`, `vi` 등의 다른 텍스트 에디터를 사용하여도 무관하다.
```
cd /root
mkdir .secret
vim .secret/cloudflare_api.ini
```

아래는 `cloudflare_api.ini`에 작성될 내용의 예시이다.  
`0123456789abcdef0123456789abcdef01234567` 위치에 앞서 발급받은 API Token을 적어준다.
```
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
```

보안을 위해 `.secret` 디렉터리의 권한과 `cloudflare_api.ini` 파일의 권한을 제한한다.
```
chmod 600 -R .secret/
```

#### Certbot Client 설치
Let's Encrypt는 Python 기반의 클라이언트로 인증서 발급에 관한 모든 절차를 처리한다.

클라이언트 설치를 위한 Python 및 pip 설치 및 Certbot 설치
```
apt install python3 python3-pip
pip3 install certbot-dns-cloudflare
```

#### 와일드카드 인증서 발급
```
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secret/cloudflare_api.ini \
  --dns-cloudflare-propagation-seconds 20 \
  -d example.com,*.example.com \
  --preferred-challenges dns-01
```
생성과정 중 도메인 관리자의 이메일 입력 (갱신 등의 정보알림용), 약관 동의 여부 및 메일링 리스트 등록의 절차를 거친다.  

인증서는 `/etc/letsencrypt/live/[domain]` 안에 저장되어있다.
`[domain]`은 인증서가 발급된 도메인을 지칭한다.

#### 인증서 자동 갱신
유닉스 계열 컴퓨터의 잡 스케쥴러인 cron을 통해 인증서를 자동으로 갱신한다.
갱신 후 웹서버를 재시작하므로(일시적인 서비스의 끊김이 발생할 수 있음), 서버의 방문자가 제일 적은 시간에 갱신을 시도한다.  

Let's Encrypt 인증서를 90일간 유효하나, 어짜피 인증서는 만료일에 충분히 가까워지지 않으면 갱신되지 않으므로, 혹시 작업이 제대로 실행되지 않아 제대로 갱신이 되는 불상사를 미연에 방지하기위해 매일 갱신 시도를 하도록 하자.

crontab을 연다.
```
crontab -e
```

열린 crontab에 다음과 같이 cron값을 추가하고 저장한다.
아래 예시는 매일 새벽 3시에 인증서 갱신을 시도한다.
```
* 3 * * * certbot renew --quiet --post-hook "service apache2 reload"
```


### Apache Virtual Host Setup
***아파치 2.4.8 이상 버전**을 기준으로 설명한다.  
여기서부터는 root게정에서 작업하지 않아도 무관하다.
관리자 권한이 필요할때만 사용하면 되므로, 안전을 위해서 root가 아닌 다른 계정에서 작업하자.*

아파치 SSL 모듈 활성화  
아래 커맨드를 실행하면 아파치 서버를 재시작하라는 메세지가 뜰텐데, 무시하자.
재시작 전에 해야할 설정이 많이 남아있다.
```
sudo a2enmod ssl
```

아파치 사이트 설정파일 저장 폴더 `/etc/apache2/sites-available` 안에 HTTPS 기본 예제가 저장되어 있다.  
이 파일을 복사 후 수정한다.
```
cd /etc/apache2/sites-available
sudo cp default-ssl.conf example.com_ssl.conf
sudo vim example.com_ssl.conf
```

- `ServerAdmin`에 서버 관리자 이메일 주소 넣기  
- `ServerName`에 도메인 이름 추가
- `SSLEngine`의 옵션이 `on`인 상태이고, 주석이 풀려있는지 확인  
- `SSLCertificateFile`에 `/etc/letsencrypt/live/example.com/fullchain.pem` 등록  
- `SSLCertificateKeyFile`에 `/etc/letsencrypt/live/example.com/privkey.pem` 등록  

아래는 수정 완료된 config 파일의 예시이다.
`...`으로 표시된 부분은 생략된 부분을 나타낸다.
```
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@example.com
        ServerName example.com

        DocumentRoot /var/www/html
        ...

        #   SSL Engine Switch:
        #   Enable/Disable SSL for this virtual host.
        SSLEngine on
        ...

        #   A self-signed (snakeoil) certificate can be created by installing
        #   the ssl-cert package. See
        #   /usr/share/doc/apache2/README.Debian.gz for more info.
        #   If both key and certificate are stored in the same file, only the
        #   SSLCertificateFile directive is needed.
        SSLCertificateFile      /etc/letsencrypt/live/example.com/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
        ...

     </VirtualHost>
</IfModule>
```

설정 파일을 다 작성하였으면, 사이트 설정 파일을 저장하고, 사이트를 활성화한다.
```
sudo a2ensite example.com_ssl.conf
sudo service apache2 reload
```

브라우저에서 `https://example.com`으로 접속하여 정상적으로 HTTPS 연결이 이루어 지는지 확인한디.


### Conclusion
지금까지 **무료**로 **신뢰할 수 있는 CA(인증기관)에서 발급한 SSL인증서**를 Let's Encrypt를 통해 발급받는 방법을 알아보았다.  
단순히 HTTPS 연결을 지원함에서 끝날 것이 아니라, 추가로 HSTS 설정(항상 HTTPS로 연결되도록 설정) 및 TLS 버전의 세부 설정 (TLS 1 버전부터 TLS 1.3 버전까지 지원), 그리고 HTTP/2의 지원(속도 향상)을 통해 더 빠르고 안전한 웹사이트를 구축해나가자.


#### Reference
1. **It's time to turn on HTTPS: the benefits are well worth the effort**: https://www.cio.com/article/3180686/its-time-to-turn-on-https-the-benefits-are-well-worth-the-effort.html (English)
2. **How to Get Let's Encrypt Wildcard Certificate with Cloudflare (Using Global API Key)**: https://blog.minase.moe/2 (Korean)
3. **Official Manual of Using Certbot**: https://certbot-dns-cloudflare.readthedocs.io/en/stable/ (English)
4. **Where are my certificates (Certbot Official User Guide)**: https://certbot.eff.org/docs/using.html#where-are-my-certificates (English)