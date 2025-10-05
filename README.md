
version: "3"
 
services:
  backend:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      MYSQL_ROOT_PASSWORD: admin
      MARIADB_ROOT_PASSWORD: admin
 
  configurator:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: none
    entrypoint:
      - bash
      - -c
    command:
      - >
        ls -1 apps > sites/apps.txt;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_QUEUE";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: db
      DB_PORT: "3306"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
 
  create-site:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: none
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    entrypoint:
      - bash
      - -c
    command:
      - >
        wait-for-it -t 120 db:3306;
        wait-for-it -t 120 redis-cache:6379;
        wait-for-it -t 120 redis-queue:6379;
        export start=`date +%s`;
        until [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".db_host // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_cache // empty"` ]] && \
          [[ -n `grep -hs ^ sites/common_site_config.json | jq -r ".redis_queue // empty"` ]];
        do
          echo "Waiting for sites/common_site_config.json to be created";
          sleep 5;
          if (( `date +%s`-start > 120 )); then
            echo "could not find sites/common_site_config.json with required keys";
            exit 1
          fi
        done;
        echo "sites/common_site_config.json found";
        bench new-site --mariadb-user-host-login-scope='%' --admin-password=admin --db-root-username=root --db-root-password=admin --install-app erpnext --set-default frontend;
 
  db:
    image: mariadb:10.6
    networks:
      - frappe_network
    healthcheck:
      test: mysqladmin ping -h localhost --password=admin
      interval: 1s
      retries: 20
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed # Temporary fix for MariaDB 10.6
    environment:
      MYSQL_ROOT_PASSWORD: admin
      MARIADB_ROOT_PASSWORD: admin
    volumes:
      - db-data:/var/lib/mysql
    ports:
      - "3309:3306"
 
  frontend:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    depends_on:
      - websocket
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - nginx-entrypoint.sh
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: frontend
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
      PROXY_READ_TIMEOUT: 120
      CLIENT_MAX_BODY_SIZE: 50m
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    ports:
      - "8082:8080"
 
  queue-long:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - long,default,short
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
 
  queue-short:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - worker
      - --queue
      - short,default
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
 
  redis-queue:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
    volumes:
      - redis-queue-data:/data
 
  redis-cache:
    image: redis:6.2-alpine
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
 
  scheduler:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - bench
      - schedule
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
 
  websocket:
    image: frappe/erpnext:v15.71.1
    networks:
      - frappe_network
    deploy:
      restart_policy:
        condition: on-failure
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
 
volumes:
  db-data:
  redis-queue-data:
  sites:
  logs:
 
networks:
  frappe_network:
    driver: bridge



[![Build Status](https://travis-ci.org/webrtc/samples.svg?branch=gh-pages)](https://travis-ci.org/webrtc/samples/)

# WebRTC code samples #

This is a repository for the WebRTC Javascript code samples.

Some of the samples use new browser features. They may only work in Chrome Canary and/or Firefox Beta, and may require flags to be set.

All of the samples use [adapter.js](https://github.com/webrtc/adapter), a shim to insulate apps from spec changes and prefix differences. In fact, the standards and protocols used for WebRTC implementations are highly stable, and there are only a few prefixed names. For full interop information, see [webrtc.org/web-apis/interop](http://www.webrtc.org/web-apis/interop).

In Chrome and Opera, all samples that use `getUserMedia()` must be run from a server. Calling `getUserMedia()` from a file:// URL will work in Firefox, but fail silently in Chrome and Opera.

[webrtc.org/testing](http://www.webrtc.org/testing) lists command line flags useful for development and testing with Chrome.

For more information about WebRTC, we maintain a list of [WebRTC Resources](https://docs.google.com/document/d/1idl_NYQhllFEFqkGQOLv8KBK8M3EVzyvxnKkHl4SuM8/edit). If you've never worked with WebRTC, we recommend you start with the 2013 Google I/O [WebRTC presentation](http://www.youtube.com/watch?v=p2HzZkd2A40).

Patches and issues welcome! See [CONTRIBUTING](https://github.com/webrtc/samples/blob/gh-pages/CONTRIBUTING.md) for instructions. All contributors must sign a contributor license agreement before code can be accepted. Please complete the agreement for an [individual](https://developers.google.com/open-source/cla/individual) or a [corporation](https://developers.google.com/open-source/cla/corporate) as appropriate.
The [Developer's Guide](https://bit.ly/webrtcdevguide) for this repo has more information about code style, structure and validation.
Head over to [test/README.md](https://github.com/webrtc/samples/blob/gh-pages/test/README.md) and get started developing.

## The demos ##

### getUserMedia ###

[Basic getUserMedia demo](https://webrtc.github.io/samples/src/content/getusermedia/gum/)

[getUserMedia + canvas](https://webrtc.github.io/samples/src/content/getusermedia/canvas/)

[getUserMedia + canvas + CSS Filters](https://webrtc.github.io/samples/src/content/getusermedia/filter/)

[getUserMedia with resolution constraints](https://webrtc.github.io/samples/src/content/getusermedia/resolution/)

[getUserMedia with camera, mic and speaker selection](https://webrtc.github.io/samples/src/content/getusermedia/source/)

[Audio-only getUserMedia output to local audio element](https://webrtc.github.io/samples/src/content/getusermedia/audio/)

[Audio-only getUserMedia displaying volume](https://webrtc.github.io/samples/src/content/getusermedia/volume/)

[Face tracking](https://webrtc.github.io/samples/src/content/getusermedia/face/)

[Record stream](https://webrtc.github.io/samples/src/content/getusermedia/record/)

### Stream capture ###

<!-- [Stream from a video element to a peer connection](https://webrtc.github.io/samples/src/content/capture/video-pc/) -->

[Stream from a canvas element to a video element](https://webrtc.github.io/samples/src/content/capture/canvas-video/)

[Stream from a canvas element to a peer connection](https://webrtc.github.io/samples/src/content/capture/canvas-pc/)

<!-- [Record a stream from a canvas element](https://webrtc.github.io/samples/src/content/capture/canvas-record/) -->

### Devices ###

[Select camera, microphone and speaker](https://webrtc.github.io/samples/src/content/devices/input-output/)

[Select media source and audio output](https://webrtc.github.io/samples/src/content/devices/multi/)

### RTCPeerConnection ###

[Basic peer connection](https://webrtc.github.io/samples/src/content/peerconnection/pc1/)

[Audio-only peer connection](https://webrtc.github.io/samples/src/content/peerconnection/audio/)

[Multiple peer connections at once](https://webrtc.github.io/samples/src/content/peerconnection/multiple/)

[Forward output of one peer connection into another](https://webrtc.github.io/samples/src/content/peerconnection/multiple-relay/)

[Munge SDP parameters](https://webrtc.github.io/samples/src/content/peerconnection/munge-sdp/)

[Use pranswer when setting up a peer connection](https://webrtc.github.io/samples/src/content/peerconnection/pr-answer/)

[Adjust constraints, view stats](https://webrtc.github.io/samples/src/content/peerconnection/constraints/)

[Display createOffer output](https://webrtc.github.io/samples/src/content/peerconnection/create-offer/)

[Use RTCDTMFSender](https://webrtc.github.io/samples/src/content/peerconnection/dtmf/)

[Display peer connection states](https://webrtc.github.io/samples/src/content/peerconnection/states/)

[ICE candidate gathering from STUN/TURN servers](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/)

[Do an ICE restart](https://webrtc.github.io/samples/src/content/peerconnection/restart-ice/)

[Web Audio output as input to peer connection](https://webrtc.github.io/samples/src/content/peerconnection/webaudio-input/)

[Peer connection as input to Web Audio](https://webrtc.github.io/samples/src/content/peerconnection/webaudio-output/)

### RTCDataChannel ###

[Transmit text](https://webrtc.github.io/samples/src/content/datachannel/basic/)

[Transfer a file](https://webrtc.github.io/samples/src/content/datachannel/filetransfer/)

[Transfer data](https://webrtc.github.io/samples/src/content/datachannel/datatransfer/)

### Video chat ###

[AppRTC video chat client](https://apprtc.appspot.com/) powered by Google App Engine

[AppRTC URL parameters](https://apprtc.appspot.com/params.html)
