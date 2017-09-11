# megabrute

-----------------
Abstract
-----------------

Megabrute is a new web assessment tool for simultaneous directory brute force on multiple targets.
This tool was developed as improvement of dirb and dirsearch for covering bigger scope at once and to be less detectable. It is good addition to nmap, masscan or shoadn.io queries results as next stage of url discovery. 

---------------------
Intro
---------------------

The idea of developing this tool was to help researchers and security experts to find weak points in the large companies web infrastructure or when needed to cover big scope of targets.

General features:
	No limitations on number of targets and payloads.
	Possibility of target classification by payloads types to optimize requests and brute force time. 
	Recover state after interruption (SIGTERM, Ctrl+c)
	Support recursion.
	Capability of using several proxies.
Input features:  
	Can use  targets from multiple sources: file, nmap, masscan, mongodb, shodan.io.
	Flexible setup list of payloads.
Output features: 
	Output is in json format. It can be analyzed on fly before brute force finishing.

---------------
Parameters:
----------------

Usage of ./megabrute:
  -T int
    	Sets timeout after each request.
  -b string
    	File which contains list of brute patterns (default "brut_patterns.txt")
  -d string
    	Sets database IP. For example 127.0.0.1:27017
  -f string
    	Path to the file with target urls.
  -n int
    	Set number of goroutines functions (number of parallel requests). Default = 10 (default 10)
  -o string
    	Sets file for output. Default is /tmp/megabrute-resalt.txt (default "/tmp/megabrute-resalt.txt")
  -p string
    	Set proxy. Example: -p ‘socks5://127.0.0.1:9050;https://123.123.123.123;’
  -q string
    	Set up mongodb query in format db_name;collection_name;find/agrigate;query;field_to_find;.
         Example: 'web;611a2eda-f8d1-11e6-9e8d-1867b0bc2b18;find;{{Final:{$exists:true}},{Final:1}};Final'
  -r int
    	Sets deep of recursion. Default = 1 (default 1)
  -t string
    	Type of url: IIS, Apache, ect. Default: Dir. (default "all")
  -u string
    	Sets target url.
  -tmp sets temporary file to store intermediate values. By default they stores in memory.
  -restore restore previous state

------------------------------------
Installation
------------------------------------

apt-get install golang
go get golang.org/x/net/proxy
go build megabrute.go

Docker file:
Coming soon .

------------------------------------
Work principles
-------------------------------------

This tool is written on Go as language that optimized for parallel tasks. So to understand some principles of go, goroutines and channels (concurrency) please read https://www.golang-book.com/books/intro/10
When megabrute starts working it align payloads types (by default from file brut-patterns.txt) and url types. If targets don’t have any type then tool sets up “Dir” type. After align tool forms loops of urls according to payloads and sends simultaneous requests. Maximum number of simultaneous requests  controlled by -n parameter.  
To reducing amount of responses and filter same or similar responds the tool calculates and  keeps in memory last 5 md5 and ssdeep hashes for each url. If a respones is unique than megabrute analyzes header to check type of server and body to find new urls for recursion.
Depends on number of targets, payloads and deep of  recursion working time of megabrute can be several hours and some times even more.  So to make possibility to analyze output in real time before megabrute will finish it continuously output results in to file. That file can be analyzed with jq tool  for example (see below). 
If in some case you need to stop megabrute you can do it by Ctrl+C combination. And tool will store it’s current state into dump_urls and dump_md5 files. On next start it will restore state from this files.

------------------------------------
Usage and recommendation examples:
-----------------------------------

Simple examples:
./megabrute -f targets.txt -n 30 -r 2
./megabrute -f “nmap:nmap-scan-rasult.xml” -b my-payloads.txt -p soks5://127.0.0.1:9040/

Customization:
Targets:
As input Megabrute can use text files, scan results of nmap and masscan tools; Shodan.io and mongodb queries.
Text files:
Text files that will be given as input should have next structure:
1) Type1,Type2,...,TypeN target_url
or 
2) target_url
where Type1,..TypeN is types of urls that should be matched to appropriate types of payloads.
target_url - is target url.

In first case urls will be sorted by the type of payloads (types of payloads and target_urls should be matched) this  lead to optimization of requests to each url by  payload type.

Text file example with types:
Dir http://127.0.0.1/files/
Mircosoft,IIS http://microsoft.com

Parsing nmap or masscan results:
For nmap and masscan results you just should put in parameter  -f before file path word nmap or masscan and ";" as separator.
Example:
merabrute -f nmap:/tmp/trget_urls

Mongodb queries:
For mongodb queries should be used two parameters:
First one -d , that sets up database url to connect to and second one is -q the query.
Query structure is the following:
db_name;collection_name;find/agrigate;query;field_to_find;
When you use agrigate the last feald "field_to_find" is not required. All parts are separated (splited) by ";" delimeter. (As delimeter used ";")
Query exmple:
megabrute -d 127.0.0.1:27017 -q 'web;611a2eda-f8d1-11e6-9e8d-1867b0bc2b18;find;{{Final:{$exists:true}},{Final:1}};Final'

Shodan.io queries:
On the way! ... (Comming soon)

Payloads
For reducing the number of request per target it is better to sort payloads by type. By default and as example megabrute use SecList patterns (sec_list_url) at brute_patterns.txt file.
Note: SecList git didn't include to this repository and should be uploaded separately.
Payload patterns looks similar to urls txt files except that instead url they have link on payloads file.
Payloads file structure is following:
<Type1>,<Type2>,...,<TypeN> file_path
Where Type1..TypeN is types of payloads that located in "file_path". And file_path is path to the file with payloads. Usually settling several types for payloads is not required.

Example:
IIS SecLists/Discovery/Web_Content/IIS.fuzz.txt
Dir SecLists/Discovery/Web_Content/raft-large-directories.txt

Note: The tool selects the types of payloads and the types of urls, compare them and than align them.

Detection and speed 
To be undetectable user should play with parameters. For this can be used parameter number of goroutines "-n" witch sets up the maximum number of simultaneous requests (which will wait for requests from server), to reduce detects better set up this value less then total number of targets. But also it depends on speed of your network and request time.
If system is very strict and still detects you. You can use timeout parameter "-T". It will sets up time to gorouting sleep after each request.  

Memory usage 
Targets loads to the memory so as more targets you have as more memory use. Payloads almost not influence on memory (usage).
On the systems with low memory you can use parameter "-tmp" to setup file with temporary data. By default found results that can be used on next stage of recursion are stored in memory. This option will force tool to store them in file.

Output parsing examples
Output is in json format so parsing can be done using jq tool. Also output it can be put in to mongo db (but it out of this tool scope). 

Structure of output:
url,md5,ssdeep hash,responds header, body, payload.
jq parsing examples:
jq '.' megabrute_results.json   #Output all results
jq 'select(.Url | contains("127.0.0.1")) | select(.ContentLength > 0) | .Url + .Payload' #Selects all urls that contains 127.0.0.1 and content length > 0. Output will be iin forrmat: http://127.0.0.1/payload

---------------------------
Enjoy!
For study cases only!
---------------------------
