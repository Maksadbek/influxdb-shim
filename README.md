# Influxdb-shim
Implementation of the shim between users and InfluxDB
Includes following functionalities:
* AUTH - user authentication via LDAP
* QUERIES - queries through to backend (InfluxDB) are limited only to those metrics that match user credentials
* LIMITS - query limiter QoS, something like between queries or max number of concurrent request
* PERMS - blacklist of commands users cannot run(```like select * from mem limit 10```)

## Configuration
The shim is configured via flags and conf file

### Flags for configuration

```
Usage of ./influxdb-shim:
  -alsologtostderr
        log to standard error as well as files
  -config string
        config file name without extension (default "conf")
  -configPath string
        config file path (default ".")
  -log_backtrace_at value
        when logging hits line file:N, emit a stack trace (default :0)
  -log_dir string
        If non-empty, write log files in this directory
  -logtostderr
        log to standard error instead of files
  -stderrthreshold value
        logs at or above this threshold go to stderr
  -v value
        log level for V logs
  -vmodule value
        comma-separated list of pattern=N settings for file-filtered logging
```

### Configuration file

Configuations are kept in toml file format, but can be changed to any other.
```toml
[auth]
    [ldap]
        name        = ""        # a name assigned to the new method of authorization
        host        = ""        # example: mydomain.com
        port        = ""        # example: 636
        useTLS      = false     # whether to use TLS when connecting to the LDAP server
        bindDN      = ""        # example: cn=Search,dc=mydomain,dc=com
        bindPasswd  = ""        # the password for the Bind DN
        userBase    = ""        # example: ou=Users,dc=mydomain,dc=com
        userFilter  = ""        # example: (&(objectClass=posixAccount)(uid=%s)), %s param will be substituted with user's username
        userDN      = ""        # example: cn=%s,ou=Users,dc=mydomain,dc=com
        useBindDN   = false     # using bindDN authentication
        attrUsername= ""
        attrName    = ""
        attrSurname = ""
        attrMail    = ""        # the attribute of the user's LDAP record containing email address, example: email
[qos]
	ttl		= 600 # TTL(in seconds) of the token for rate limiter
	limit   = 100 # count of queries allowed during the TTL
[influxdb]
    addr        = "127.0.0.1:8086"
    username    = ""
    password    = ""
    userAgent   = ""
[web]
    addr        = "127.0.0.1:8888"
[blacklist]
    queries     = [""]        # blacklist of queries that is prohibitied to run, example: "SHOW DATABASES"
    adminGroup  = "admin"     # admin group name, this group members can see & run everything
# group specifications, group members have allowed and denied query list
[[groups]]                      # [[groups]]
    ou = ""                     #     ou = "Global group"
    cn = ""                     #     cn = "Admin"
    queries = [                 #     deniedQueries = [
        "",                     #         "Show measurements",
        ""                      #         "SHOW TAGS"
    ]                           #     ]
```

## Usage
To use the shim users must firstly get JWT token string.
The purpose of chosing JWT token is that it already contains the required info about user in itself.
That is why no need to keep user info in backend.

#### 0. Generate private & public keys for JWT generation
```bash
# generate private key
$ openssl genrsa -out app.rsa 1024
# generate public key
$ openssl rsa -in app.rsa -pubout > app.rsa.pub
```
#### 1. Authorize with your LDAP credentials and get token string 

Initially the shim listens ```8888``` port

**Input parameters:**
* ```uid``` the user id from LDAP credentials
* ```p``` the password from LDAP credentials

```
curl -XPOST "localhost:8888/auth?uid=tesla&p=password"
```
This request returns the token string
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InRlc2xhQGxkYXAuZm9ydW1zeXMuY29tIiwiaXNBZG1pbiI6ZmFsc2UsIm5hbWUiOiJ0ZXNsYSIsInN1cm5hbWUiOiJUZXNsYSIsInVzZXJuYW1lIjoiTmlrb2xhIFRlc2xhIn0.d_VhNIDcQ9qYMv2gbmq-8VypHw-dr2N3UWds-JMEkj2KBzFujYBzQV2G03grNPi5wR6AdwBESdys3vZNZJ1edK9LyH1dC4KsBL7xhMfmOR4dW1IMNoc_3C7BW1oWKat8Mu0-3rHmA7fftlfr5aKBs2uhDqLtuVSG28731-3g7L8
```


#### 2. Send queries with 

**Input parameters**

* ```q``` InfluxDB query
* ```db``` InfluxDB database
* ```AccessToken``` in header is the token string

Header of the request must include ```AccessToken```, it must include the token string that is receipt from authorization 

```
curl -G 'http://localhost:8888/query?pretty=true' \
    --data-urlencode "db=mydb" \
    --data-urlencode "q=SHOW TAGS" \
    -H "AccessToken: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6InRlc2xhQGxkYXAuZm9ydW1zeXMuY29tIiwiaXNBZG1pbiI6ZmFsc2UsIm5hbWUiOiJ0ZXNsYSIsInN1cm5hbWUiOiJUZXNsYSIsInVzZXJuYW1lIjoiTmlrb2xhIFRlc2xhIn0.d_VhNIDcQ9qYMv2gbmq-8VypHw-dr2N3UWds-JMEkj2KBzFujYBzQV2G03grNPi5wR6AdwBESdys3vZNZJ1edK9LyH1dC4KsBL7xhMfmOR4dW1IMNoc_3C7BW1oWKat8Mu0-3rHmA7fftlfr5aKBs2uhDqLtuVSG28731-3g7L8"
```
