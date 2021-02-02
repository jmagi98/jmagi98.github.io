# Introduction:
One attack surface that has always interested me is IoT devices. I feel like there is a little something for everyone, whether you want to break out the soldering iron or Burp Suite. Recently, I decided to try my luck with a $30 dollar "smart" lock. The lock allows you to open it either through it's fingerprint sensor or a mobile application paired to it over Bluetooth. Lets just say it was way less secure than even I would have thought.

## Disclaimer:
This post does not contain the name or manufacturer of the lock, nor does it contain any screenshots, exact code, or exposed endpoints. While I don't think I have done anything wrong, I don't really feel like testing the waters. My research was conducted using my own account(s) and was done strictly for the purposes of testing the security of the product. If you are interested in testing this PoC out yourself, just google around on Amazon for "Fingerprint smart lock" and you may be able to find something similar. This lock is not widely used at the time of writing this, and the company/individual producing them has no contact information or avenue for disclosure. If you are looking to secure your home or other valuables with a product along these lines, my first piece of advice is **don't**. If you absolutely must, then do so from a reputable company. Lastly, this research was conducted individually on my own time. 

Okay with that out of the way I think we are good to go now.

# Recon:

After receiving the device from Amazon, the first thing I did was download the associated application from the Google Play Store on a rooted test device. I specifically mention this was a "test device" because the app asks for A LOT of permissions. Camera, files, media...pretty much everything. I would not recommend putting this on anything that has your real data. The next part of my research will likely be to dig more into the APK itself and see if there is anything malicious happening under the hood.

## IDOR Vuln:
After signing up for an account, I used `adb logcat` in order to see if any logs had been left in production. There were tons. First, I was able to see the endpoint the application was reaching out to. Second, my `secret_token` was logged. This secret token was used as a header in the requests to the API. The application itself logged the full request, and I noticed something strange. While the endpoints themselves were called things like `/getAccount`, the device was communicating to them via post request. The body of the post request looked similar to`{"user_id": 123456789}`...Do you see where this is going?

# How the Application is Intended to Work:
Before I get into the exploit itself, I want to talk about the intended functionality of the app. After you log into the application, you are able to see a list of locks you are authorized to unlock. In addition, there is a friend feature. Person A can send a friend request to Person B. Once the two are friends, they have the ability to grant the other authorization to open their locks. In concept, this would be kind of cool. A use case may be a family wanting each member to be able to open a lock from their individual phone. However, paired with the insecure API, there are some serious issues.

# Exploit:
After discovering the weird API calls the phone was making, I decided to make a second account. I retrieved the `user_id` for that second account via the logs. Then, I crawled the application using Burp Suite by walking through the entire application, one action at a time.

Taking a look at the results, I confirmed that every call to the API was a POST request, and the body of that request always included the user's id. Furthermore, the only form of authentication that was included, was the `token` value included in the header. That token was different across my two accounts, but remained static for each API call.

As you probably have guessed by now, changing the `user_id` parameter in the body of the POST request resulted in the corresponding new user's account being affected. I was able to retrieve my second account's details knowing only the id, and nothing else. I was still using the `token` for the first account, but there was no cross validation to ensure the token itself and the `user_id` were actually associated. The token simply granted full authorization to the API. With that in mind, I was able to script up an exploit in Python. The steps of the exploit are as follows:
## 1. Send a friend request to the victim.
This does require having the victim's email, as the body of the request uses this to lookup the user. However, even if you did not know the email, you could technically enumerate every single user using the `getAccount` functionality, and just changing the `user_id`. When you send a friend request, the response will include the `user_id` of the victim who you are attempting to befriend. This value will be used for the rest of the exploit chain.

## 2. Have the victim user accept the request.
When a friend request is accepted, a POST is sent to the API with a body similar to `{"p_id": 123456789,"user_id": 987654321, "status": "2"}`. The `p_id` is the value of the user who has received the friend request, and the `user_id` is the value of the person who sent the request. Knowing the victim's id and our own id, we can send a POST to this API endpoint that looks like `{"p_id": VICTIMS_ID,"user_id": OUR_ID, "status": "2"}` and accept the request on their behalf. Now we are friends with the user.

## 3. Grant access to the lock.
For the grand finale, using the victim's id, we are able to send a request and see the locks the have associated with their account. From there, we can get the `lock_id` of the lock we want to access. Finally, we can authorize ourselves to access the lock by sending another POST request with the body `{"lock_id": 1234, "p_id": VICTIMS_ID, "user_id": OUR_ID}`. Now, using our phone shows the lock in our lock list, and we are able to connect to it and unlock it, having known nothing but the victim's email.

# Conclusion:
When I bought this device, I thought I knew what I was getting myself into. I did not expect, however, to find a vuln so quickly. And I certainly did not expect it to be as serious as this one. The script I have described above used a total of 6 API endpoints, but these are certainly not the only (or most catastrophic) ones. Again I want to emphasize that this whole PoC was done on two accounts I own myself, but the vulnerability affects every user equally. I stress that if you **must** buy "smart" products, please do so from a reputable company.
 
