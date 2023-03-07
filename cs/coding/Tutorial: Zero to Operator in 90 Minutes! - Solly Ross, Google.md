Join us for Kubernetes Forums Seoul, Sydney, Bengaluru and Delhi - learn more at kubecon.io  Don't miss KubeCon + CloudNativeCon 2020 events in Amsterdam March 30 - April 2, Shanghai July 28-30 and Boston November 17-20! Learn more at kubecon.io. 

The conference features presentations from developers and end users of Kubernetes, Prometheus, Envoy, and all of the other CNCF-hosted projects  

Tutorial: Zero to Operator in 90 Minutes! - Solly Ross, Google (Limited Available Seating; First-Come, First-Served Basis)    

- Please bring your laptop fully charged as we will have limited charging stations available in the room.    

- Please complete the following steps ahead of time to make your tutorial easier:   [https://gist.github.com/DirectXMan12/...](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbGI1dmFwRmwzd1ZXNXMwUnZxWXQ0Q0RPRVNGQXxBQ3Jtc0ttMFcxN1NOUG9aa0lQQ1QwQ2F3LU1hczhFa1piZnEyVDRYb0V5YlF2WVlQdDV0Y00wOUlVeGJYQk9MSXN5TXkySmVYeDh3b0k2dmczWmtlQmJ0TXpjZnZCVTJZQ3AwOGg1dmZEV0trRTVtNkhPbkFyaw&q=https%3A%2F%2Fgist.github.com%2FDirectXMan12%2Fad7b35327c2816125a45cdc11ff78476&v=KBTXBUVNF2I)   
- Come learn how to quickly get off the ground running with building an operator using KubeBuilder v2!  
- Come write a Kubernetes-style API to manage a bespoke application, complete with declarative validation and defaulting. 
- Discover what kind of requirements go into an API type, and how to write API types that work and feel like they're part of Kubernetes, and can be easily consumed as part of a larger system.  
- Once you've got an API type, you'll make use of the new server-side apply functionality to make implementing your core logic a breeze, and learn how to think about writing well-behaved controller logic that deals with different interactions with other parts of Kubernetes.  
- Finally, you'll learn how to actually run your controller locally for development and on a remote cluster for production.  



Introduce background and basic setup

0:00

all right so before we get started if you are having issues with like setting up your own cluster or if the conference Wi-Fi is being frustrating please raise

0:14

your hand my coworker over here will come along with a GCP account information and you can use that GCP count to create VMs and run some stuff

in VMs so I'll talk a little bit more about doing that in a bit it's still a little quiet ok all right so welcome to zero to operator thank you all for coming today for those of you who don't know me I'm Solly Ross I'm a software engineer on GKE and a maintainer of kubebuilder and my mission is to make writing kubernetes extensions less arcane 

so before we get started let's quickly talk about some definitions so to define an operator we have to define a controller so a controller is a loop that reads desired state observed cluster state and potentially some external state so things like load balancers on your cloud for instance and then reconciles the desired state with  the observed cluster States and the observed external state and writes down observations at the end so that other things can see what we're doing this is a relatively uncontroversial definition all of kubernetes functions on this model everything from the deployment controller down to the node even the scheduler

 ok so now that we have that in place let's define an operator an operator is a controller that has specific knowledge of an application so for instance the Prometheus operator knows how to manage Prometheus specifically it knows how to take a bunch of rules from different namespaces about how to monitor applications bundle them up into something that actually can be passed to Prometheus itself as configuration and manage some number of Prometheus instances so **all operators are controllers but not all controllers are operators** some controllers just have general knowledge so the deployment control art doesn't have any specific insight into what it's managing all it knows is that it needs to do some rolling updates 

all right so generally over the course of this presentation 

- I'm going to talk to you about some concepts 

- then we're gonna look at some practical details 
- and then I'm gonna pop over and try it out and y'all can follow along and try it out as well 
- so we'll talk about scaffolding and design 
- and then we'll move into types 
- and then we'll move into the code that makes those types do something useful 
- and then finally at the end we'll talk about moving from your test dev loop into something that can be deployed on an actual cluster for production use 

all right so without further ado perhaps a little to do if some of you think you have seen similar things before it's probably true I've given some similar presentations before but in this one we're gonna talk a lot about the new features in kubernetes that make this even nicer some of the stuff that just came out recently right so kube-builder is a set of opinions and building blocks so there's kind of three components:

- there's kube-builder itself which is scaffolding out new projects 
- and then there's that's built on top of two libraries the first is controller runtime which is like a standard library for writing controllers and the second is controller tools which is a set of tools to do stuff like compile your go types into a custom resource definition that you can install in the cluster so that you don't have to write open API spec by hand because that is not a fun experience in my opinion 

alright so today we're going to be building an operator for the guestbook tutorial from kubernetes so who here has played around with the guestbook tutorial before at all alright so cool some of you it's a pretty simple application there's a small front-end PHP application that connects to a Redis instance and then just basically lets you write in a name stores the name in Redis and when you load the page and loads all the names from Redis pretty simple but it has a few moving parts and so it'll be good to explore kind of some stuff on how to write an operator so we're gonna manage and deploy both and then we're going to expose the whole application in terms of a service alright so there is a repo for the finished code for this tutorial since we have an hour and a half I'd recommend y'all take a look at that that might help you save you some time typing it is at present dev slash cubic on u.s. 2019 slash code and I am going to give people a few minutes now to pull that up

5:49

also if you put chopped off the slash code you can follow along with the slides if you ever want to take a closer

5:56

look at something so this is all just on my website once you're in that repo

6:02

there's a directory called goals that is like the Amol from the tutorial so if

6:10

you want to look at like the stuff that we're going to generate on the cluster you can take a look in there so give

6:20

like 30 seconds for people to copy down that URL

6:39

all right so I sent out homework ahead of time hopefully at least some of you

6:47

who've done the homework but if you haven't you're going to need a copy of Q

6:52

builder you're going to want a copy of the complete repo probably and you might

7:01

want to pull open the original tutorial in your web browser just to take a look

7:07

and it might help to compare and contrast what we're doing in the operator so

7:25

oh yeah you can totally you can brew

7:30

install if brew has to 2.0 you can totally install from brew

7:37

yeah we have a couple of contributors that maintain a brew repository at this

7:44

point you're also if you don't have a cluster up and running you're also going to want to kick off launching a cluster

7:52

so I'm just going to quickly flip over to my terminal for those of you don't

7:59

have one and I'm just going to show you what I did so I already have one up and

8:07

running for the sake of this tutorial but so we can see I have created this VM

8:18

here it is a container optimized OS

8:23

image because we're just going to be running a cluster on it for the most part if you can't get other stuff

8:29

running on your system you might want to do is the Debian image instead so that you can install other tooling but I'm

8:35

just using it to run a kind cluster so I will just briefly show you what I did is

8:43

we see if we test the internet right now

8:55

cool right so once largest there we go

9:04

awesome so for those of you who don't have a cluster I'd recommend using kind so the first thing you're going to want

9:12

to do is download kind which is here if

9:18

you go to the kind web page so kind dot cigs gates do you can get this very long

9:24

URL so don't try to copy it from here just know that you or if you google

9:31

kubernetes times you will find the right web page and then if you're on container optimized OS you just need to move

9:38

kind to VAR run and then make it

9:47

executable like that and then finally

9:53

you can do VAR run kinds create cluster

10:01

and kick that off it'll take a couple minutes I'm gonna pop back to the presentation but it's good to get it

10:09

started now okay cool

10:14

on with the show so how do we get

10:19

started with queue builder so what we're gonna do is we're gonna make a new directory since this is a go project

10:25

we're going to initialize the go module in it and then we're going to scaffold out the basic queue builder code this

10:32

will create some initial project structure creating directories for our API types and for our controller and

10:40

give us some nice deployment files for later on we give it this domain here this is just the suffix that is put on

10:49

the end of our API groups so that we don't clash with anyone else alright so

10:55

I'm gonna do that now up over here C

11:01

will create a new in progress you all

11:08

see that let me make it bigger go mod

11:15

and knit coupon you can name this go module whatever you want and then we'll

11:22

do queue builder in it - - Jemaine I'm

11:28

gonna use my personal domain please don't use this domain this one's mine

11:33

you can choose whatever you want if you have your own domain name use that you

11:39

could use example.com if you really want to it's reserved for testing so right

11:44

and it's just going to scaffold out a project with the default dependencies

11:50

it'll hint at our next step if you want spoilers you don't want spoilers ignore that last line okay cool

11:58

so ugly alright so I'm you heard me

12:04

mention API groups a moment ago so let's talk about how we describe kubernetes api sand will howl explain what API

12:10

groups are so in kubernetes we collect API types together and so we call each

12:17

collection of related API types in API group API groups can have multiple

12:23

versions this is how we make breaking changes to api's in kubernetes we add

12:30

new versions and then we convert between them and within that API group we call

12:35

each type a kind now this is most of what you need to know to talk about

12:40

kubernetes api terminology but there is one more thing that may come up if you get really deep in you may also hear

12:47

people talk about resources it resources a use of a particular kind than the API

12:53

usually they're one-to-one but not always there's a few exceptions in certain parts of the kubernetes api so

13:01

just you can think of resources as functions and kinds as return types but

13:07

usually in kubernetes we only have like one set of functions that returns a given return type and so when we have go

13:14

types usually each of those go types corresponds to a single group version kind all right so the structure of every

13:25

almost every kubernetes kind looks roughly like this so all of them start

13:30

out with some metadata this provides information about what kind we're

13:36

dealing with what API version in this case we're looking at the pod in the

13:41

core v1 group we for historical reasons we don't write the core but it's kind of

13:47

implied and then there's also some what we call object metadata so this is things like name namespace labels

13:53

hopefully y'all are familiar with that and then almost all other types almost

13:59

have spec and status so the spec contains the desired state and

14:05

the status contains the observes date right so practically speaking the way we

14:14

get started here is we call queue build or create API we pass a group name and

14:19

then I'll get our domain appended so if I say group web app it'll be web app doesn't matter magic hole depth we give

14:26

it a kind name so we'll say guestbook here and then we give it a version we're

14:31

gonna be bold and we're gonna decide that we're going right to stable protip don't do this and that'll give us

14:39

something that looks like this for our API types so you can see up top here we have our spec and then we have our

14:47

status and then we have this root object which contains our metadata so type

14:52

meant an object meta so that's like API version and name and namespace and stuff

14:58

and then it also contains our spec in our status and finally we have this list type as you might guess this is the type

15:06

that's returned when we call west's so if we do cube cuddle get without a specific name it goes and says to the

15:13

API server give me all of the pods and the API server returns this kind of

15:18

record type all right so let's look through each of those sections so first

15:23

up is the root type we don't usually edit the go code of the root type because it almost never

15:29

changes between types but what we do do is we can add additional metadata to the

15:37

root type this describes more information about our CR D so the common

15:42

things that we do is we enable sub resources and we add print columns so we'll talk about the status sub resource

15:49

a little bit later in the tutorial but for now just know that we enable it here with this marker comment so that's that

15:57

comment that begins with a plus and then we also had print columns print columns are the way that we can get nice cube

16:04

cut' will get output so you know like with core types if I were to do cube cut' will get pods I would see like some

16:10

nice columns that would show like node name and maybe pot IP and stuff like that status whether or not it's running or

16:17

terminated or in progress the way we do that for CR DS is we had these print columns so here we've added to print

16:25

columns we add one for URL which we're gonna be exposing in our status and we

16:32

had one that shows the current desired replicas from our spec and this way when we type cube cut' we'll get guest bucks

16:37

instead of just seeing name and like creation timestamp we see like some useful information that is a summary of

16:44

all of our guestbook instances right on to the spec so spec describes desired

16:53

state like we said before so we need to decide what we want to put in here so I've put in two kind of fields in the

17:01

root of the spec one of them is the name of the Redis object that we're going to use to fetch our connection details so

17:08

we'll also have a Redis kind and we will use the Redis status to know how to connect our deployment to Redis and then

17:16

we also have details about how to deploy and manage our front-end so I've added resource requirements so we can control

17:23

how much CPU it uses and also information about the service and the number of replicas and you can see we

17:28

also can use markers here marker comments this time they go in fields and

17:34

they control stuff like defaulting and validation so we actually have a pretty powerful language and open API and all

17:40

of that is exposed to these markers so we can say don't let people submit ports

17:45

that are negative because that doesn't make any sense if they don't add a port make it 8080

17:51

because that's a same default for instance and we can say that certain fields are optional so like port is

17:57

optional will default it all right status is very similar to spec there is

18:04

one thing to note here as you're thinking about your api's which is to say status is used to convey information

18:10

to the user and to other controllers it is not for storing things so you should

18:17

think about status as kind of ephemeral right like imagine that at any time someone could come through and delete

18:23

your entire status you need to be able to recreate it from everything else so a given

18:28

controller that's writing status to its own object should not be reading in that status

18:33

it should not be depending on information there it's it's for it to communicate information out to the world

18:39

right so we're also going to need Redis types kind of similar to what we talked

18:46

about for guestbook Redis has some one

18:52

leader and some number of follower instances so we let you control the

18:57

follower replicas a couple things to note here we can use Joe doc to set

19:04

basically API descriptions that come out in our open API this is nice if people are generate using nice UI is that or

19:12

IDs that give them help when they're writing their mo and it also shows up in cue cuddle explained so if someone says

19:18

cute cuddly explained guest books it'll read from the open API and I'll show like follower replicas is the number of

19:25

follower instances to run so right so if

19:31

you were paying super close attention you might have noticed that when I default 'add two fields in the guestbook

19:38

I use slightly different types for them so one of them was a pointer and one of

19:43

them was not a pointer this is a little bit of an important point in kubernetes so if we don't have a pointer the zero

19:52

value is assumed to mean unset so in the case of integers for instance that means

19:59

zero is the same thing as I didn't fill in a value and so the default will

20:05

always be applied even if the user passes zero so this is great for things like port because zero is kind of a

20:12

nonsensical port but it's not so great for replicas because it means we can

20:17

never stop our thing right so the solution to this is to make the zero value something else and the way we do

20:23

that is by using a pointer so if the zero zero value of a pointer is nil

20:28

which means unset becomes nil becomes a default but if it's explicitly set to

20:34

zero in the mo that becomes a pointer to zero and we don't do our defaulting so we let our users scale down our

20:41

things which is good because sometimes they won't want it running we can use

20:46

almost any type in go in our spec and status with a few exceptions

20:52

we don't use floats because for those of you not familiar with how floats

20:57

interact and they're they're very good at what they do but that what they do is probably not what you expect so they

21:03

don't round-trip very well through through Jason and llamó and through other languages so we use a different

21:08

representation instead called resource stock quantity you've probably seen it when you're sending your pod resource

21:14

requirements we also don't really use interfaces in our api's we prefer these kind of tagged unions down here where we

21:20

have an explicit type field and then multiple kind of variant fields that are

21:25

generally pointers to things or things with zero of all values all right so

21:32

let's try this out so we're gonna populate our scaffold it out sample so

21:40

by default queue builder scaffolds out kind of an empty sample with some bogus data in it so we can fill that out I'm

21:46

going to add some bad data that's an invalid field into mine just so that we

21:52

can see that validation is working and then we will actually pop over and

21:59

generate our CRD from our go types with make manifests we will install our CR D

22:05

with cube cuddle create and we will attempt to install our sample it will fail and then we will install it again

22:14

and it will succeed because we will remove the bad data and we'll be B we

22:20

will be able to run cube cuddle get guests books and it will work so let's

22:26

try that out just gonna flip back over

22:32

to over here so first I'm gonna fill out I'm gonna do all that stuff we talked

22:37

about at the beginning so we'll do Q builder create API we're

22:44

gonna get a API and it's gonna ask us do we want to create a resource we're gonna say yes that's do we want to create

22:50

those go types and do we want to create a controller we'll also say yes we'll deal with that in a little so it's going to scaffold out some types

22:59

that's going to attempt to run go imports to clean it up a little bit and then we've got this the scaffold out

23:07

types so I am going to cheat a little bit because I'm standing up here and I'm

23:12

gonna copy over from the completed tutorial so API v1 guestbook types and

23:20

then we'll also look at you know API v1

23:26

guestbook types in this not split so you

23:32

all can see both sides we can see we've got this empty here we've got this

23:37

complete here so I'm just going to grab my spec fields from here and copy them

23:44

over get rid of this nonsense and then

23:51

you can see we have like this nested front end spec so we'll also need that and then finally our status with just

24:00

the URL we could put other information in here too but oftentimes we have stuff

24:07

like status conditions that convey kind of what the current state is of the

24:13

deployment of the thing so like did we create our service successfully did create our deployment successfully and

24:21

then finally I'll also copy in all of those markers that go on our route type

24:29

over here and then finally last but

24:37

certainly not least since we use that core v1 dot resource requirements that's the same kind of structure used in the

24:44

pod container resource requirements we'll need to import core v1 and then my

24:50

compiler is happy so I'm all set I can run make manifests oh one more thing

25:00

sorry about this so we're using brand new features and so we've got a tell

25:08

controller gen that we want to generate in the new format with those new features so this was just launched in

25:15

kubernetes like one version ago and so by default we don't generate that so that people get projects that work on

25:21

older versions of new kubernetes but here we like living on the edge so we're

25:26

gonna say use the GA version so we open up the make file and say CRT versions

25:32

equals v1 as opposed to v1 beta 1 and let's try that again cool alright so

25:39

let's take a look at that quickly just so you can see that all the stuff you didn't have to write so we'll do config

25:45

CRD basis and you can see we get this nice open API spec that has all this

25:52

validation in here we get some defaulting you can see we had that

25:58

faulting on replicas with the default of 1 in the minimum of 0 and you can see

26:05

that we have all this nice resource requirements stuff in here too so let's

26:11

also edit our sample so it's under the config samples guestbook and we'll give

26:18

it a Redis name since that was required by this sample will generate the Redis

26:24

one later later and then we'll give it some front-end configuration so I'll say

26:30

resources requirements or requests sorry

26:36

say CPU 50 ml of course it's not a big application and then we'll also say

26:44

serving port whole and negative one and

26:49

that's just invalid right you can't have a porch of negative one and so our validation will catch that all right so

26:57

at this point all of you should have running kind cluster if you were on GCP

27:03

it should be up and running and if you were not on GC p your local cluster

27:08

should hopefully be running or your remote cluster or whatever whatever you're deploying on to so what I did

27:18

because I'm running against GC p is that I as saged into the node the compute instance

27:25

I grabbed the cube config that was generated by kind so kind we'll say it's your cube config is now set it's at

27:32

Tilley slash dot cube slash config so I copied that to my local machine and then

27:39

I'm going to run this fancy port forward command and I will show you how to

27:45

figure that out in a moment so swell commands over here I'm just gonna grab

27:53

this thing here so basically this is just the default g-cloud ssh command

28:00

that you will get if you press the little drop down next to the instance box and say next ssh and you say ssh

28:06

with g-cloud and then we're gonna pass this dash out here and - f + - n which

28:12

is going to port forward from my local machine to the remote machine and so

28:18

we're just going to use the connection string we got from the cube config so I'll show you how to get that in a

28:24

moment but we're just gonna run this here - F and - and just say no I don't

28:31

actually run it and want to run a command just go into the background in port forward and with any luck we should

28:39

be able to do cool so we have a working cluster right so let me just show you

28:50

console clouds at google.com and we'll go under our compute engine instances

28:58

here and then so the way I got that crazy command was I just went down to

29:05

this little drop down here and I said you g-cloud command and you just copy this whole thing and then at the end of

29:13

it you just add - - and then - L and the

29:20

port and then the connection string from the cupid favour

29:28

please if anybody is having issues with that or is confused please raise your hand and we can have someone come over

29:39

all right so now that you hopefully have a running cluster let's install the CRD

29:48

right and we'll also attempt to create our sample and we're gonna get this nice

29:58

message here see so our validation is working it said invalid value spec front

30:07

end that serving porch should be greater than or equal to zero so we're gonna go into config samples

30:15

and get rid of serving porch here and then we were going to try again it works

30:23

because we provided value valid data and then if we do cube cut' will get guestbooks we can see we have this nice

30:33

table display that shows us useful information just because we put those

30:39

markers in place we don't have any URL because we don't have any controller to set the URL so let's tackle that I'm

30:47

gonna give everybody a couple more minutes first though to try this out and remember all of this stuff is in the

30:55

completed get repo so if you are having

31:01

trouble or you think you type out something you can't figure out what it is or if you just don't feel like typing I frequently don't feel like typing

31:08

that's why I have Kay Elias to cue cuddle instead of having to type out cube cut' all the time you can just go

31:14

from there and if you really don't want spoilers in that repo I've got the exact

31:21

git commit where you should be checked out to in the completed repo if you

31:27

really want to just follow along exactly instead of just using the end right and

31:34

feel free to raise your hand if you are having issues

31:51

and or if you think there's something that's very confusing you can also ask

31:57

the question out loud and I will try to answer it on stage because it may be confusing for other people as well would

32:09

you say slow down a little bit yeah I can I'll give you I'll give you all a

32:15

few more minutes and as a reminder if

32:26

you would like to look back at any of the other slides if you go to present

32:31

meta magical dev which I can show you now you will get this webpage if the

32:42

conference Wi-Fi is working and you can just click on this link here that says 0

32:48

\- operator a little zoom in a little bit this link here that says 0 - operator and you'll actually just get a copy of

32:54

the slides so if you would like to look at the slides at all or look back a few slides or whatever you can follow along

33:01

there as well can also look forward a few slides but that would be spoilers again and that's no fun

33:35

if you and if you want to ask a question you can also come up here and I'll repeat the question out loud if if it's

33:40

like you think it's relevant for everyone

33:50

actually I'll come down and help somebody dull to just expedite this Hey

34:20

it could make my own

35:15

yeah so I've got a couple of good questions the first one is what was that

35:21

make file option and it's CRD versions plural equals v1 and you can see that in

35:28

the completed repo too and the other one is if you get this weird error about go

35:35

dependencies that occasionally happens

35:41

due to the way go likes to resolve dependencies if you go into your Godot

35:47

mod file you should see if you're encountering this issue you should see two lines one that says controller

35:53

runtime and one that says kate's io / client go if you delete the line that says kto / client go and then you rerun

36:02

like go build or go mod tidy or something like that basically any of the commands that effect go modules it will

36:10

attempt to reira solve dependencies and actually use the right version basically kubernetes go dependencies management is

36:20

like half way to go modules and so occasionally it gets a little bit weird oh that changes in Godot mod so it's

36:31

your Godot mod is your dependency less right it's like your version of your all

36:37

your modules that you're using

37:05

the question was about the new features we're using in our CR D and what what

37:14

versions will this work with and and why are why are we turning on this feature so this is actually about things like

37:22

defaulting so those are only turned on in the GA version of CR DS and generally

37:33

you could hypothetically write a controller that assume the defaulting

37:40

was not going to take place and like back up defaulted but use the CR d

37:46

defaulting if it was available and just generate two different versions of the CR D that's actually why the option is

37:53

plural you can actually specify multiple versions there and install a different version depending on what cluster it's

37:58

running on that's kind of a little bit more advanced than this tutorial I think

38:05

so we're just gonna stick with stuff for the newer version

38:42

when you do queue build or create API it'll ask you two questions say yes to

38:48

both of them

38:59

yeah so actually if you had that failure with the dependencies it's just failure

39:07

attempting to build I believe you should have all the files still there okay so

39:14

if you then deleted them you can go into your project file it's just called

39:19

project and all upper case and there's like a list of types and if you just delete it you should be able to run the

39:26

command again

40:15

just a reminder so you will need a 1.16 or newer technically if you're compiling

40:22

from head a cluster so that's why we're using kind today for instance anything

40:30

older you'll get messages about the v1 CRD not existing or if you try to

40:36

generate v1 beta one it'll yell at you and say you're not allowed to use defaults in v1 beta one

44:41

okay so make sure if you're missing

44:46

config CRD bases make sure to run this first command make manifests that will

44:53

generate your CR DS and put them in that bassist directory that's probably it

44:58

alright quick show of hands who has managed to successfully run Q cuddle get

45:05

guest books this last step all right who has like gotten mostly the way there and

45:13

like hit some speed bump okay and who is

45:19

still waiting for their cluster to finish creating because of Internet issues okay

45:26

so for the sake of time just to make sure we get through everything I'm going

45:32

to advance along a little bit and then if we have time at the end or the next break I'll be happy to come around again

45:38

and and help out and Walter can also come around and help you out alright so

45:47

we have our types and this is awesome but at this point kubernetes is effectively acting acting as a

45:53

structured key value star which is like great but we probably actually want to

45:59

do stuff right deploy pods run workloads all that jazz see URLs have things show

46:06

up in our browser that's the fun demo endpoint right alright so let's talk about how we make things go so when we

46:15

talk about kubernetes and we talked about controllers the core concept that we talked about is reconciliation

46:20

reconciling our observed and desired states and so generally in kubernetes

46:26

reconciliation follows like a standard pattern this is followed by almost everything in kubernetes be it custom

46:33

operators custom controllers built-in controllers even like the qubit so we

46:39

read our route object the main object we care about for any given controller we

46:44

fetch all the other objects we care about so these are might be objects we created objects we're reading

46:50

information from we ensure the objects we created are in the right State we insure that our external state things

46:57

like load balancers databases etc are in the right seat and then we write back

47:02

what we did in to our object status and then we repeat over and over again so

47:08

let's take a look at what that looks like in code form so this is a little small but we'll go over it in more

47:15

detail and I can zoom in a little bit but this is in the essence the entire

47:20

reconciliation loop for our guestbook controller so we can see we start out we

47:27

get this reconciliation request here and that contains the name in the name space for a guestbook

47:34

so what we're gonna do is we're gonna attempt to load that guestbook and then

47:40

once we've loaded that guestbook we're also going to attempt to load the Redis

47:45

instance that it references so we're gonna look at spec dot Redis name and

47:52

we're gonna construct a key basically a name in the namespace and we're gonna say load the Redis name spectra Tasneem

47:59

in the namespace that's the same as our guestbook and then using information

48:05

from that Redis is status and our guestbook spec we're going to construct the deployment to run our guestbook

48:12

front-end and then we're also going to use the

48:17

information in our guestbook spec to construct the service that exposes that

48:22

deployment to the world and then finally we are going to submit those changes to

48:28

the API server and update our own status and then update our the value of our

48:36

status on the API server and finally we return this result value that result

48:42

value says things completed successfully and we don't need to riku so if we

48:48

needed to do periodic updates or if we need to check again later because say we're reconciling some external status

48:56

that doesn't implement this kind of watch mechanism that we have in kubernetes we could say like riku after

49:03

10 seconds to check again in 10 seconds well we don't need to do that also if we return an error here as the second value we

49:11

would get automatically recued with back off so if we fail to do something we can just return an error and our mechanics

49:18

will try again later but backing off so that we don't just keep trying really

49:24

quickly forever all right so you might have noticed there were a couple helpers there I kind of moved

49:31

them out so that it would be easier to go over the structure of the code but the helpers are actually pretty simple

49:37

so normally in kubernetes we need to be

49:42

make sure to be item potent with our changes all right so what this means is if our object is in the right state

49:49

already we don't want to accidentally reset any fields that someone else set

49:54

or change things we just want to make sure that everything is in the right state that we care about and so

50:00

previously in older versions of kubernetes this was kind of hard if any

50:06

of you were have seen any of my previous presentations you'll know that there's

50:11

like a couple pages of code just to do this this has gotten a lot nicer with a

50:19

feature called server-side apply which I like to call the Picard patch so what server-side apply lets you do is

50:26

it lets you declare the object in UML or Jason form and you just set the fields

50:33

that you care about and you always set all the fields that you care about and then you tell the API server make it so

50:40

and the API server figures out what changes if any need to be made to the

50:46

actual object and this allows multiple writers to make their changes and just

50:52

say ensure it's the way I want it without stomping over each other so here even if say the service

50:59

controller populates some more information or add some defaulting in even though we haven't specified that

51:04

defaulting we're just constructing it from scratch every time we won't clobber out those changes and so you can kind of

51:13

see this go code almost just looks like we've written kubernetes Hamill and go and that's basically because we have

51:19

it's very similar to how cube cut' all apply works but way more powerful because multiple

51:24

writers can do it and it's on the server so we don't need any special logic client-side view one other thing that

51:30

we're doing here is we're setting an owner reference this does two things the first is that when we delete our

51:37

root guestbook object it'll make sure that our service and deployment are also cleaned up this is called garbage

51:43

collection in kubernetes if you want to google for more information about it the other thing it does is a little bit

51:50

later on when we go to wire up our reconciler so that it's actually called it let's our controller machinery know

51:58

when you see changes for this deployment it's actually we need to be reconciled

52:06

instance the other one here is pretty similar it's how we construct our status

52:13

information you can just see we load the service we set some fields we always set

52:19

the value so if we don't have any information we explicitly say that and if we do we say that okay

52:26

so now we just need some wiring we need to tell it when to run we have our logic in place but it's not gonna do anything

52:33

so when we talk about wiring kubernetes controllers we talk about relationships

52:40

between objects so each controller is

52:45

for a single kind it can have other kinds that it cares about but there's

52:52

this kind of route kind that the reconciler functions in terms of and so our case for a guestbook controller the

52:58

route kind as you may have guessed from the name is guestbook and then we talked about two types of relationships one is

53:05

a very common relationship and that's the owns relationship this is for objects we create so we define that

53:12

relationship in terms of that owner reference that we just set and then there's another relationship which is

53:17

more free form called watches this is for objects we didn't create we probably don't modify but we do care about some

53:25

values in them so in the case of our guestbook if we flip back a couple slides we can see that we pull

53:31

information specifically we pull up we actually don't have we'll see

53:40

in a moment we pull information for constructing our deployment that was just the service from the Redis status

53:48

so we care when the Reta status changes because if it changes service names we

53:54

need to know so we can update our deployment configuration so we say that guestbook watches Redis alright so what

54:02

does this look like in terms of code practically so you can see basically our

54:08

reconcile our logic goes in this method called reconcile that implements the reconciler interface basically what we

54:16

saw before is everything in the reconciler the only difference is we set up some context and logging information

54:22

just so that we can see what's going on and then we put all of that log code that we just wrote in the middle here

54:28

should look fairly familiar all right and then we need to register that

54:35

reconciler with a controller that knows how to trigger it so we say we want to

54:41

create a new controller managed by a manager the manager is just responsible for setting up watches and all the

54:48

boilerplate and running our controllers and waiting for signal handlers all that like boring stuff that actually just we

54:55

don't really care about what we need to do for everything and then we say that this controller is for guestbooks

55:02

it owns services and deployments just like we talked about and it watches Redis instances now watches as you can

55:09

see is a bit more involved because it's an arbitrary relationship we need to specify what that relationship is and we

55:17

do that using a mapping function now the mapping function needs to be able to map

55:22

from a Redis instance back up to a guestbook and so to do that we need to

55:28

be able to quickly look up what guestbooks reference particular reticent sense and so we do that by setting in

55:34

index you can see this index is pretty simple basically we just say what is the

55:40

Redis name and then we index on the Redis name well see we'll use that in a moment in our mapping function so the

55:46

mapping function is also pretty straightforward basically the meat is this function

55:52

right here so all we're doing is we're saying given some particular Redis

55:59

instance list all of the guest books but

56:05

filter on that index we just set up so that we only get the guest books that

56:11

match that have spec dot Redis name as whatever the thing we're reconciling is

56:18

and then once we get all the guest books we just convert them into these reconcile requests and all that is is

56:24

just a name in the namespace so all we do is we go through each of the guest books we pull out the name of the namespace and we put it into this

56:32

reconcile request object and we return it and then our controller pushes all those onto the queue to get reconciled

56:40

all right so that was a lot so we're

56:46

gonna actually try that out now what I'd recommend is that you copy and paste

56:53

that from the completed repo you totally can take the contents in the

56:59

presentation and copy them in or write it out by hand but I'm assuming most of you like me are a little bit lazy and

57:05

wouldn't want to type all of that deployment go object out so I would recommend copying and pasting it and

57:12

then when once we've done that we can do we can run make run and that will run

57:18

the controller locally against our remote cluster this is super useful for

57:23

dev test cycles because we don't have to bundle it into a docker image and then deploy it on the cluster wait for it to

57:29

deploy and we can even do stuff like run it under Dell to debug if something goes terribly wrong so we're just going to

57:36

try that out so I am going to go and copy over the code myself and then I'll

57:42

come down and answer questions from y'all

57:59

yes sorry about that - remember what the

58:07

directory was called okay cool all right

58:18

let me make that bigger cool so I'm just

58:23

going to go into controllers here and we see we have this guestbook controller

58:29

and it kind of looks like what we just saw we have this empty reconciler so I'm

58:34

gonna split with the completed one and

58:40

we're gonna grab the complete controller which is effectively just what we went

58:48

over before all right I ignore some of

58:53

this extra stuff on top we'll talk about that in a bit I'm just going to copy go over of the

59:01

internals of this scaffolding here copy all of this stuff which is just what we

59:07

saw exactly and I'll paste it back in here and then same thing with our setup

59:18

here and I've actually moved that help

59:23

those helpers that we saw out into their own file just for organization purposes

59:30

so I'm going to copy that over we're also going to need all of the core API

59:37

types that we referenced so let's put those in place - right

59:45

so basically these two are the kubernetes api types for deployment and service and these three are the bits

59:54

that we need for writing that watches relationship all this says go doc so if

1:00:01

you're interested you can go and look at the go doc as well it's at Godot org so

1:00:08

I SIG's a kate's that I have slash controller runtime there's a link at the end of the slides too

1:00:13

all right so this is currently going to complain if I had to guess that we don't

1:00:19

have any of these helper methods and we don't have a Redis instance so I'm going

1:00:24

to really quickly create the reticence since - I'm gonna copy that over I

1:00:29

forgot to do that earlier beats

1:00:46

that'ss - version the one - - group web

1:00:53

app just like we did know just like we

1:01:00

did for our guestbook it's gonna complain then I'm gonna copy over all

1:01:06

the stuff that we're missing and then

1:01:16

we'll copy over that helpers file which is exactly the code that we just talked about under controllers cool and so now

1:01:35

if any luck hey Rach look we should be

1:01:45

able to do make run

1:01:52

let's regenerate for

1:02:07

yeah that's why sorry copied some stuff incorrectly let's fix that fix that up

1:02:14

perils of live coding right so we called

1:02:23

this Q con instead of workshop code and the this version so we just need to make

1:02:29

this cube con right let's try that again

1:02:37

cool so we'll install our Redis CRD as well since we have it I haven't done

1:02:44

that yet I don't know if any of you have but I haven't done it well make sure

1:02:51

cube cuddle is on my path and then we'll do that cool now we should be able to

1:02:57

connect run

1:03:02

aha we can see we've got this log line that says successfully reconciled our

1:03:10

guestbook controller so we have a running controller for guest bucks a

1:03:18

little bit later there's also code in the repo for the Redis controller I did

1:03:25

not cover it in the presentation because it looks almost exactly the same as the guestbook controller instead of creating

1:03:31

one deployment in service we create two deployments and services that's basically it

1:03:38

alright so let's take a look at the time cool I'm gonna give you all some time to

1:03:46

try this out if you are confused or unclear about anything or something's

1:03:51

not working as always raise your hands

1:04:00

we just turn it

1:05:58

just to reiterate if you're looking at the complete repository you'll also see those Redis types I just mentioned and

1:06:05

the Redis controller so for the final full working thing you'll also need the

1:06:11

Redis controller if you've got the guestbook controller up and running and

1:06:16

you're curious you can look at the Redis controller but it should look very very

1:06:21

very similar to the guestbook controller we walk through we create two deployments one for the leader one for

1:06:29

the followers we create two services one for the leader one for the followers and then we write down the leader and

1:06:36

follower service names in our status

1:06:56

but again if you're getting that error with like wrong number of arguments and

1:07:02

watch it's basically due to pulling to new of her version of client go so if

1:07:09

you go into your Godot mod you should see a number of lines that start with just cait's dot IO it should be API

1:07:16

machinery and client go and API if you just delete those and then run make run

1:07:22

go we'll reevaluate the dependency graph and find the right versions

1:07:56

this is actually a good point to bring up so modifying our controller does not

1:08:02

change our CRD definition so once you

1:08:07

add the Redis types in the only thing you need to reinstall hold on is the

1:08:15

Redis types the guestbook types haven't changed from when we generated them before even though we've changed the

1:08:25

controller itself the controller does not affect the CRD types

1:08:47

and one other note if you are running in kind do system last-minute changes that

1:08:55

URL is not going to show up because it's waiting to see a load balance service so

1:09:01

we could and I will if you want if we have time modify the service status code

1:09:07

to instead of looking for a load balancer vez or waiting for a load balancer vez just do a normal service

1:09:12

and put say the kubernetes api prefix in there instead but it's basically the

1:09:20

same premise

1:09:45

as a quick as a quick show of hands for a status check how many of you have

1:09:52

gotten make run running right how many

1:09:57

of you are still copying code how many of you have not gotten to this slides

1:10:05

point yet and are still trying to work out stuff from the last pause okay and

1:10:12

last but not least how many of you are encountering issues with make run cool

1:10:19

all right I will try to come around if you have issues with make run especially

1:10:26

raise your hand male to try to come around and figure out what they are and hopefully we can take care of some

1:10:31

issues for the group

1:11:18

so if you are running locally you are docker pulling over a conference Wi-Fi

1:11:24

on your cluster this may take a while

1:11:30

yes you should see a deployment all right

1:16:28

okay so if you copy code over from the completed workshop make sure you change

1:16:34

the import path so like my import path for the example workshop code was

1:16:42

workshop - code as the base but if you did go mod in it mycube con you're gonna

1:16:49

have to replace workshop - code with my cube con for instance

1:17:40

for those of you who are not seeing a

1:17:46

set of deployments as I am we have a

1:17:51

quick for this and this is cuz I was lazy and did not add sufficient logging to my controller so this is a

1:17:59

lesson to y'all make sure you have good instrumentation so let's uh let's figure

1:18:12

this out this is a good good learning opportunity so if something's going wrong in our guestbook controllers it's

1:18:19

not creating our deployments I'm going off the script here let's hope it works

1:18:25

alright so something's gone wrong so the first thing we probably want to do is we want

1:18:31

to add some logging so let's do this we'll say got Redis now I'm cheating a

1:18:39

little bit because I have a very strong idea of what's going wrong wrong and then we'll give a controller runtime

1:18:46

uses structured logging by default so we can cast key value pairs after our message so in this case we're gonna say

1:18:53

got Redis and then we'll give the Redis name so Redis name like that alright so

1:18:59

now we're gonna run make run again and we should see that we don't actually see

1:19:06

got Redis alright so I had to guess

1:19:21

or we got an empty rebus so let's try that again I'm guessing that we're

1:19:33

hitting an error there yeah see there we go didn't guy read us and that's cuz we

1:19:40

never created a reticence since whoops so let's edit our Redis sample

1:19:56

there we go that was rather quick all we did is set the follower replicas to one I think we set a default for that by

1:20:03

default so we didn't even need to do that but we'll do it and then we're gonna do all right let's try that again

1:20:17

and in fact you should

1:20:30

there we go so we see we got this got reddest line and we can actually see this is kind of cool with our structured

1:20:36

logging we have these key value pairs here and we see the ret we got the Redis ready seam so let's try that again tada

1:20:50

alright we actually have something that works now cool so I'm going to move on

1:21:03

because we're running a little bit low on time and then it's a very end I'll be

1:21:08

happy to come around and help anyone else who's still having issues all right

1:21:14

so before we are done we need to talk

1:21:19

about how to package this up and run it on the cluster so there's two things we

1:21:24

need to talk about and the first of which is permissions so right now I'm guessing y'all are running the

1:21:31

controller as cluster admin this is not great do not run all of your controllers as

1:21:37

cluster admin I can safely say this without being a security person so the

1:21:44

way we control permissions in kubernetes is generally through something called our back role based access control our

1:21:51

back functions in terms of roles as you might have guessed and bindings of users

1:21:56

to those roles so by defaults we generate a role binding that binds the

1:22:01

default service account that's like a user for a controller in the namespace

1:22:07

we're running to this role that automatically gets created by queue builder queue builder scaffolds out

1:22:15

markers to generate rules for that robot for that role that let us read our own

1:22:21

types but since we're also reading and writing other types we need permissions to read those too so we'll add these two

1:22:28

markers in here so this one says for the group apps and the resource deployments

1:22:35

we're gonna allow lists watch so that's so we can list and watch and trigger

1:22:41

reconciliation yet so we can fetch and patch patches the

1:22:48

the underlying verb behind server side apply same thing with services we need

1:22:53

to get list watch and package so that'll generate these roles and then when we run make

1:23:00

manifest it will regenerate right so the other thing that we probably want to

1:23:07

talk about is running this properly so

1:23:14

cube builder scaffolds out make file rules to build and run your controller

1:23:21

as a container on the cluster as a deployment in fact so this first command here will actually

1:23:28

build all your go code and then it will

1:23:33

push it up to a registry so for those of you who have GCR or GCP accounts you can

1:23:40

push to GCR if you'd like so you got GCR slashed your project name slash and then

1:23:48

whatever you want and then we should be able to deploy and that'll just launch a

1:23:56

deployment on the cluster with that image that runs our controller and if all went rel well and we're running both

1:24:03

the reddest controller and the guestbooks controller we should actually have a functioning guestbook instance

1:24:13

alright so i'm going to switch back here

1:24:18

and just for quickness is sake i'm going

1:24:23

to switch to the complete one because that's got everything all set and then

1:24:30

i'm going to cheat a little bit

1:24:35

i don't actually want to do all these doc our operations in my laptop because

1:24:40

it's gonna be slow so i'm going to run them with Google cloud build which is

1:24:50

like docker builds but on GCP so well that's running I'm gonna come

1:24:59

around and see who else has questions right as a show of hands who were able

1:25:07

to who was able to run make run from the last step who's still stuck at like the

1:25:13

make run portion alright and who's stuck before the make run portion okay cool

1:25:21

hey keep your hands up and I will come

1:25:26

around to help out

1:25:56

under at this point you can basically just go to the end of the completed repo

1:26:03

so like where you originally cloned or you can go to that sha at the bottom

1:26:08

there but basically we're at the end at this point

1:27:27

anybody is all finished and wants to do some further reading or investigation

1:27:32

you can go to book Q builder dot IO and there's other tutorials there and

1:27:39

whatnot just in case any of you have to scoot now or whatever and want to see

1:27:44

that before I advance to the next slide

1:32:45

this were roughly out of time I'm happy to continue staying around helping people out but I figured I'd advance the

1:32:52

last slide there are some resources up here if y'all want to learn more and

1:33:00

thank you all for coming out



英语 (自动生成)

###  