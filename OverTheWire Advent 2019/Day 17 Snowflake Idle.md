# AOTW 2019, 17th: Snowflake Idle


* Points: 482
* Solves: 22 
* Author: hpmv
* Category: web, crypto

> It's not just leprechauns that hide their gold at the end of the rainbow, elves hide their candy there too. 

## Task

The webpage looks roughly like an idle game. As a button says, the task is to "Buy a flag with 1 vigintillion (10^63) snowflakes". Obviously it won't work by "collecting snowflakes" and "upgrading collection speed".


## Recon

After clicking a while the amount of flakes showed floating point errors like python does in e.g.:
```python
>>> 0.1+0.2
0.30000000000000004
```
This will be helpful later.

After going through all the functionality with burp, there are 3 different URLs visible:
* /client
* /control
* /history/client

A JSON file is delivered on /history/client (which is used to draw a graph via javascript), e.g.:

```json
[
[1576882894816,{"action":"melt"}],
[1576882896078,{"action":"state"}],
[1576882897008,{"action":"upgrade"}],
[1576882898466,{"action":"state"}],
[1576882904682,{"action":"collect","amount":4},
...
[1576882928116,{"action":"buy_flag"}]
]
```

Sending buy_flag obviously didn't work.

Increasing the amount in action:collect didn't help in getting more flakes but big numbers where reflected in the JSON above. Still not actionable.

I tried fuzzing a bit using burp which showed the message, that the challenge is not about sending lots of requests and they're limited to 1/sec. After throttling burp it showed some interesting results like the "id" cookie gave 200 on a even number of pipes (URL encoded) and 500 on an odd number. But wasn't actionable.

The end of recon came after the idea to check if there's also a /histoy/control. This showed a similar history of actions like /history/client:

```json
{"action":"save","data":"ePk3OZeri1P/fsLt/HW4iD9Xu6Bmv3hs2f/eUecwkrG3Z66IP0G9pDD5Jw=="}
{"action":"load"}
{"action":"save","data":"ePk3OZeri1P/fsLy6nytkSQd8vw64mNvwPfKR+l+0a+iIPHMPx7r9i/7eDiYo5dT/37RuaQkp4pg"}
{"action":"load"}
```

It was possible to send action:save in the /control URL and thereby changing the amount of snowflakes in the account to any state in the past. 

Because the category was also crypto, I spend some time trying to decrypt the base64 using xor but with no success.


## Solution

Getting any past state doesn't help in getting more flakes, so I observed how the b64-blob changed as the amount changed by using the collect + melt buttons. These are the blobs of 3-6 flakes, just 2 digits change:

3. ePk3OZeri1P/fsLv/H2tkSQd8vw64mNvwPfLSOl+0a+iIPHMPx7r**9i**/7eDiYo5dT/37RuaQkp4pg
4. ePk3OZeri1P/fsPy532tkSQd8vw64mNvwPfGROl+0a+iIPHMPx7r**8S**/7eDiYo5dT/37RuaQkp4pg
5. ePk3OZeri1P/fsLy63GskSQd8vw64mNvwPfARul+0a+iIPHMPx7r**8C**/7eDiYo5dT/37RuaQkp4pg
6. ePk3OZeri1P/fsPy6nGnkCQd8vw64mNvwPbBRul+0a+iIPHMPx7r**8y**/7eDiYo5dT/37RuaQkp4pg

(Upgrading the collection speed changed digits near the end of the blob but that didn't look as promising.)
In other samples only 1 digit changed. Looked like the amount of flakes was stored as a string in the blobs, so I tried iterating the base64-charset in the positions that changed when collecting or melting flakes. At that point I had 1207.084138561806 flakes and managed to change the dot into a number which gave 1207**4**084138561806 flakes using this script:

```python
#!/usr/bin/python3 -u

import requests

b64="ePk3OZeri1P/fsLy4HWjnC0c//Qw429gyPjHSaB1wur+ZbbbbUGuoSHhemTJ4tJTqz+eufB/tIp4Uqr2IaY="
position=40

URL_POST_SAVE = 'http://3.93.128.89:1217/control'
URL_POST_STATE = 'http://3.93.128.89:1217/client'

b64_charset="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789/+"


cookie="PHPSESSID=5e4e4c3592a08600094d7988503e0f32; id=yQzAB7E1zWeZBjX5QElU0EJsz1i1fgUwcR0P2vL1tqQ="

def save(session, b64):
    url = URL_POST_SAVE
    print("postting URL save: " + url )
    resp = session.post(url,  json={"action":"save", "data": b64 }, headers={'Cookie': cookie} )
    print("---------------------------------------------------------------------------")
    print(resp.headers)
    print(resp.text)

def state(session):
    url = URL_POST_STATE
    print("postting URL state " + url )
    resp = session.post(url,  json={"action":"state" }, headers={'Cookie': cookie} )
    print("---------------------------------------------------------------------------")
    print(resp.headers)
    print(resp.text)
    if "e+96" in resp.text:
        sys.exit()

for char in b64_charset:
    session = requests.Session()
    b64_tmp=b64[:position] + char + b64[position+1:]
    print("doing: " + b64_tmp)
    save(session, b64_tmp)
    state(session)

```

Great success, but still far off 1e63. So hopped I could change it to 12074084138561e06 but that didn't work (probably because it would have to change to +e (1206510000000000+e34, so I would have to iterate through 2 characters at once.) While thinking on how to get on I clicked sometimes on collect and upgrade and at some point the amount changed to 1.2074084138561658e+16, hurray. Now I only had to change the e16 to e96 using my script above at the right position: 1.2074084138561658e+96 ... and I could buy the flag with the button:

Flag: AOTW{leaKinG_3ndp0int5}

