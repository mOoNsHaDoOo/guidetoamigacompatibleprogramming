# The Communication between tasks on Amiga


*"Message passing is faster on ami cause it just passes
around pointers to structs[...]"*
Tom "Phantom Lord" Kennedy, comp.sys.amiga.advocacy, May 5, 1994

*"copy-on-write message passing would be really great on the Amiga,
since you could get memory protection"*
Gavriel State, comp.sys.amiga.advocacy, Feb 26, 1992

*"AmigaOS definitely cannot, at least not without breaking existing
software. 
That's because an arbitrarily large chunk of memory (or even many of
them, if you pass a list) may be "attached" to the message structure 
and would have to be copied to the receiver's address space. But the 
OS doesn't know how much data actually belongs to your message. 
Most programmers don't care much about the mn_Length field, and even 
if they did it wouldn't work for lists."*
Matthias Bethke, comp.sys.amiga.programmer, Dec 18, 1996


## 5.1 Different kinds of interaction

So far we just took care of writing programs that identify themselves
inside the system as a little universe, with its own rules and its own
borders. The only external interation has been the one through graphic
feedback, in our example we cited several times the button pressing, that
is recognized from our program and is handled in a certain way. The
feedback was finally shown to the user.
We also treated in other terms the problem of the interaction between
user and program; instead now we are going to look at the interaction
between the program, the system and other programs.


## 5.2 An introduction to signals

The Amiga system warn a task about a determinate event through signals.
A signal is represented by one or more bits inside a 32-bit field, and
this field is assigned to each task. Each task has an overall of 32
signals, with 16 of them reserved to the OS.
We already met and interacted with some of the reserved signals, in
particular in the paragraph 4.5.2 we made our application able to
interact in case 4 reserved bits of our task were changed:

```c
while (DoMethod(app,MUIM_Application_NewInput,&sigs) != MUIV_Application_ReturnID_Quit)
{
  if (sigs)
  {
    sigs = Wait(sigs | SIGBREAKF_CTRL_C);
    if (sigs & SIGBREAKF_CTRL_C) break;
  }
}
```

Those 4 bits are shown with the symbol SIGBREAKF_CTRL_C. The generic
change of our task mask is instead reported inside the sigs integer,
using the MUIM_Application_NewInput method. Through the Wait()
function, instead we tell the system to send our task in Sleep state
and to wake it up only in case of the mask SIGBREAKF_CTRL_C or any other
mask inside sigs are detected. So far, the described cycle workflow
should be quite clear: until MUIM_Application_NewInput is returning a
different value than MUIV_Application_ReturnID_Quit (the window close
gadget associated value) will check the signal mask put in sigs.
If sigs value is different from zero then the Wait() function is invoked
so that the task is sent in Sleep and will be awaken only if the
same mask inside sigs is detected or in case the one shown by
SIGBREAKF_CTRL_C is detected.
In general, in case we would like to send our task to Sleep and wake it
up only if the signal 3 is received, we should use something like:

```c
Wait(1L<<3);
```

Through the left shift, the value 00000000000000000000000000000001
becomes 00000000000000000000000000001000,that is the "3" signal.
Each task (struct task) keep inside several signal masks; is important
to note the mask used to contain the allocated task signals
(tc_SigAlloc) and the mask used to keep the signals that the task might
wait to receive (tc_SigWait). Therefore a task might allocate signals,
giving them a meaning of its own, and eventually wait for their coming.
That is the reason which is said that a task signals concerns the task
itself.


## 5.3 Ports and Messages

Lets suppose we have two running tasks, A and B. The A task needs to
communicate some informations to task B; we need a communication
standard to make data from A have a meaning for B and the reverse.
Most operating systems solves this problem giving to the programmer an
indirect communication through messages and ports (IPC).
In indirect communication, the task A "writes" a message with the
required data for B and sends it through a communication port.
We can compare the communication port to a Mailbox of someone, the task
B in this case. Once the message reaches the destination port, it can be
taken by B and used for its own internal processes.
In some cases the target task is waiting for a message in a specific
port, in some other cases the recipient is waiting after the message
has been sent to a port; the latter is called synchronous
communication.
The reverse, meaning, sending and receiving messages does not include
waiting, is instead called "Asynchronous communication".
It can also be possible that a notification of the message arrival can be
requested, and sometimes might be needed to also notify the completion of
the actions made through the data contained in the received message.
All Amiga OSes support the described cases through a series of
functions offered by Exec.


5.3.1. Messages on Amiga OSes

A message, in Amiga OSes, is composed by a header and a body.
The header the message is handled by the system, while the body
of the message can have any kind of format and shape.
In other terms, an Amiga OS message is a simple structure composed with
a mandatory field and other field made by programmer choice.
In this example, if we need to compose a message containing an integer we can
write:

```c
struct MyMessage{
  struct Message msg;   
  int myInteger;
};
```

In this case the message body is represented from our integer, while the
struct Message is system header and it is still constant.
The last one is composed like this:

```c
struct Message {
  struct Node  mn_Node;
  struct MsgPort  *mn_ReplyPort;
  UWORD mn_Length;
};
```

The first structure field is used to hook the message to a port. The
second field, sometimes not mandatory, is filled with a pointer to an
answering port in case there is the need to answer with the same datas
(or different datas) to the recipient task. The third and last field,
instead, shows the message length.

In Amiga OS's messages are passed between tasks through addresses, meaning
that when the task A creates and sends a message to task B, this one
will receive a pointer to the message, not a copy of it. In a similar
situation, therefore, the message exchanging is nothing other than a way
for a task to have granted access to a chunk of allocated memory and,
apparently, to another task.
This implementation is obviously accidental to the protected memory
implementation (see par. 2.6.1), besides that this IPC implementation
has been kept in the newer Amiga OSs in order to keep some
compatibility with the older Amiga OS 3.x apps.
Amiga OS 3.x use often the IPC system and, therefore, implementing a
different system than IPC on newer Amiga OSes means that older Amiga OS
3.x applications cannot run in those systems, unless through
complicated emualtion systems.


### 5.3.2  Creation, search and removal of Port in Amiga OS's

In the last paragraph, one of the fields in the Message structure was
represented by a pointer to MsgPort structure, it is the message port
representation in the Amiga Os's:

```c
struct MsgPort
{
    struct Node mp_Node;
    UBYTE       mp_Flags;
    UBYTE       mp_SigBit;  
    APTR        mp_SigTask; 
    struct List mp_MsgList; 
};
```

The first field, mp_Node, is the header and through it the port is added
to the system lists handled by Exec; the last field, mp_MsgList
contains a pointer to the messages list queued inside the port. The
mp_SigTask field anchors the pointer to the task which the port belongs,
and this one represents the destination task, which will need to be
warned of the incoming message through the signal mask mp_SigBit.
Anyway, it is possible to modify the port behavior at the moment of
receiving a message; to obtain this, the field mp_Flags is available;
in a generic case it assumes the special value PA_SIGNAL; instead, in
case the message arrival has to come unnotified, the field will be set
to PA_IGNORE.
Usually, the message port fields are not allocated by hand, beside
particular cases.
The generic case uses the function:

```c
struct MsgPort *CreateMsgPort( VOID ); 
```

which takes care to allocate the port and its fields automatically
setting, in the example, the field mp_Flags to the value PA_SIGNAL and the
field mp_SigTask to the corresponding task.

A port can be private or public. If our needs are focusing us on the
second option, we should then act by hand on two mp_Node subfields
belonging to the allocated port with CreateMsgPort(). In detail, we
will be forced to assign a symbolic name and a priority to our port so
that it can be added and recognized unequivocally from the system.
For example, if we would like to allocate the public port Pippo, we need
to write:

```c
struct MsgPort *fooPort = CreateMsgPort();
 
fooPort->mp_Node.ln_Name = "Foo";
fooPort->mp_Node.ln_Pri = 0;

AddPort(fooPort); 
```

Doing so we assigned the name "Foo" to the port and set its priority
to zero.
To add this port to the list of the ports handled by Exec we then use
the function call:

```c
VOID AddPort( struct MsgPort *port );
```

In case of a public port we could look for this one using its symbolic
name that identifies it in the Exec lists. This information is used as
a parameter for the function:

```c
struct MsgPort *FindPort( CONST_STRPTR name );
```

Besides that, we need to highlight the fact that a task doesn't have complete
control over public ports; in fact these might be changed or removed
without notice; therefore the use of a public port has to be put in a
privileged condition or, in other words, the port use code has to be
put in a code block delimited by the system calls Forbid() and
Permit(). In particular, the validity of a public port cannot in any
way be guaranteed after a Permit() call and, therefore, extreme caution is
needed.

Once we used our public port, we need to remove it from the Exec's
Public ports list. To do this we use the function:

```c
VOID RemPort( struct MsgPort *port );
```

Last, to remove the port from the memory, either a public or private
port, we will use the function:

```c
VOID DeleteMsgPort( struct MsgPort *port );
```


### 5.3.3 - Amiga OS's Message Handling

The Message Handling on Amiga OS's can happen synchronously or
Asynchronously, using in what is supposed the best way, the following
functions:

```c
VOID             PutMsg( struct MsgPort *port, struct Message *message );
struct Message * WaitPort( struct MsgPort *port );
struct Message * GetMsg( struct MsgPort *port );
VOID             ReplyMsg( struct Message *message );
```

PutMsg() is used when it is needed to send a message to a port. The message
sending might involve the loss of control of the CPU (context
switching) from the recipient task, which would give the resource to
the receiving task.
In case that from the recipient task an answer is requested, in the field
mn_ReplyPort, situated inside our message header, will be inserted a
pointer to an eventual port, which will be used as "answer box". 
If we need to do so, the mn_ReplyPort field will need to be configured
before PutMsg() is called.

WaitPort() allow a task to wait for an incoming message in a specified
port. The task will be set on Sleep if no message is coming on the
specified port and will switch to execution in case of an incoming message
to the port. In this case WaitPort() will return the pointer to the
message; by the way this function will not take care of the removal of
the message from the port. To do that, we will use the GetMsg()
function.
GetMsg() let's us obtain the pointer to the first message inside a port with
no wait. If the port contains no message, GetMsg will return the NULL
value. If messages are present, GetMsg() will remove the first received
message from the port, returning its pointer.

ReplyMsg() is used in the moment the ongoing operations of the receiving
task is completed. If for example, the receiving task, once obtained the
message, executes some operations for which the received message
contained the required data, the receiving task will notify the completion
of the operations to the recipient task through a call to ReplyMsg().
ReplyMsg() is also used to give back the control to the message (that
is nothing other than a memory block) to the recipient task, so that
this one can deallocate the resources used by the message. In other
words, a synchronization is made between recipient and receiver, using
for the purpose the same message processed by the receiver, that will
be given back to the recipient task through ReplyMsg().
ReplyMsg() uses the mn_ReplyPort field inside the message header, where
the function expects a pointer to a port in which to send back the
message.


### 5.3.4 IPC synchronous communication step-by-step

Now let's go back to the beginning problem: we have two tasks, A and B
that need to communicate to each other. The communication will happen
through the exchange of one or more messages. In order to have all this
working, every task will have its own port, that will call Port_A and
Port_B.

Let's suppose that task B has defined a symbolic name for its Port_B,
and made it a public port.

Now, task A wants to send a message to task B and starts then to prepare
one, putting the message length in mn_Length and a reference to Port_A
in mn_ReplyPort.

Then task A looks for Port_B through its symbolic name using FindPort().
As we already know, a public port can be accessed and modified in any
moment from a task; therefore the FindPort() call must be included in
a Forbid() / Permit() block.

Now task A can send its message using PutMsg(), which will require
as the first argument the pointer to Port_B returned by FindPort() and the
pointer to the previously prepared message.

At this point the task might decide to wait until the message reaches the
receiver and that it will finish the operations involving the
message.  In order to achieve this, the function WaitPort() is used,
followed by a call to GetMsg(), which will remove the message from the
port. The calls to WaitPort() and GetMsg() might be enclosed in a loop,
depending on how many messages the two tasks will need to swap; right
now we are considering one message exchange only.

So, right now we have a similar situation inside the task A
corresponding program, consider the example in the message struct
MyMessage:

```c
struct MyMessage
{
  struct Message msg;
  int myInteger;
};

...

{//communication part
  
  struct Process *this = (struct Process *) FindTask(NULL);
  struct MsgPort *Port_A =(struct MsgPort *) &this->pr_MsgPort;
  struct MyMessage myMsg;

  myMsg.myInteger = 3;
  myMsg.msg.mn_Length = sizeof(struct MyMessage);
  myMsg.msg.mn_ReplyPort = Port_A;

  Forbid();
  {// Forbid()/Permit() block

    struct MsgPort Port_B = FindPort("Port_B");

    if (Port_B)
      PutMsg(Port_B,(struct Message *) &myMsg);
  }
  Permit();

  WaitPort(Port_A);
  GetMsg(Port_A);
}          
```

The most perceptive readers might have surely noticed the instructions:

```c
struct Process *this = (struct Process *) FindTask(NULL);
struct MsgPort *Port_A =(struct MsgPort *) &this->pr_MsgPort;
```

As we should know by now, a task is a structure containing a reference
to a task, generic in this case; in an application as those shown in
this guide, once we execute our program it will be shown in the memory
as a task. Calling FindTask() with a NULL parameter will simply return
the pointer to our program task but, since our program IS a process, then
all the fields of the Process structure just initialised will be valid.
Of course, in order for this procedure to work we need to be sure that our
program in memory is a process, but this happens in the majority of
cases... A process includes a message port assigned at its creation,
therefore the second instruction is also valid.

Now, if everything worked fine the task B has received on its Port_B our
message. For Your Information at this point the task B has several ways
to handle the message queue in its port (and on others eventually).

Now we are going to show the procedure offered by MUI.


### 5.3.5 MUIM_Application_AddInputHandler and MUIM_Application_RemInputHandler

The notions from the previous paragraphs helps us to be
able to solve the problem of two tasks communicating between each
other.
MUI programming adds to the knowledge learned so far the possibility to
encapsulate the IPC communication processed according to OOP criteria
using for that purpose the MUIM_Application_AddInputHandler.
Basically, the method MUIM_Application_AddInputHandler allows us to
handle an eventual answer from task B inside our own method instead
of bloat the main cycle of our application B with extra controls (see
paragraph 4.5.2).

The method hooked to the Port_B through MUIM_Application_AddInputHandler
will return TRUE if it accomplished succesfully all of its operations, otherwise
obviously it will return FALSE.

Let's first define in the data area of our own MUIC_Application subclass
something like this:

```c
struct appData
{
  ...

  struct MsgPort *Port_B;

  ...

  struct MUI_InputHandlerNode appHandler;   

  ...

 };
 
MUI_InputHandlerNode is a supporting structure composed as:
 
struct MUI_InputHandlerNode
{
    ...

    Object        *ihn_Object;

    union
    {
      ULONG ihn_sigs;
      
      ...
    
    } ihn_stuff;

    ULONG          ihn_Flags; 
    ULONG          ihn_Method;
};

#define ihn_Signals ihn_stuff.ihn_sigs
```

Let's enhance the fact that:

- ihn_Object    :   will contain the pointer to an object, in example
                    instantiated from our MUIC_Application subclass;
- ihn_Signals   :   will contain the associated signals to our Port_B
                    (mp_SigBit);
- ihn_Method    :   will contain the symbolic reference to our method
                    assigned at message receiving;
- ihn_Flags     :   will conatins a value that will notify MUI each
                    time we will need to control a port, usually when set 
                    to 0;

Under those conditions, usually, inside our method OM_NEW, will we have
something that looks like:

```c
static IPTR mNew(struct IClass *cl,Object *obj,struct opSet *msg)
{  
  [INITIALISATION]
  ...

  struct appData *data = INST_DATA(cl,obj);
  struct Process *this = (struct Process *) FindTask(NULL);
  data->Port_B         = (struct MsgPort *) &this->pr_MsgPort;             

  memcpy(data,&temp,sizeof(*data));                         

  ...

  data->appHandler.ihn_Object  = obj;
  data->appHandler.ihn_Signals = 1L<<data->Port_B->mp_SigBit;
  data->appHandler.ihn_Method  = MUIM_appSubClass_MyReplyMethod;
  data->appHandler.ihn_Flags   = 0;
 
  DoSuperMethod(cl,obj,MUIM_Application_AddInputHandler,(ULONG)&data->appHandler);


  ...

  return (IPTR)obj;
}  
```

The description is quite simple: in our example it is only missing the
implementation of our method MUIM_appSubClass_MyReplyMethod, that might
look like:

```c
static IPTR mMyReplyMethod(struct IClass *cl,Object *obj,struct MUIP_appSubClass_MyReplyMethod  *msg)
{
  struct appData *data = INST_DATA(cl,obj);
  IPTR ret = 0;

  if (data->Port_B)
  {
    struct MyMessage *mex;

    while((mex = (struct MyMessage *)GetMsg(data->Port_B)) != NULL)
    {
      if([CHECK IF VALUES INSIDE THE MESSAGE ARE VALID])
      {
        ...
      }
      
      ReplyMsg((struct Message *)mex);        
      ret = (IPTR) TRUE;
    }
  }

  return ret;
}        
```

Then, in the closing phase, inside the OM_DISPOSE method we will remove the
link that we made, using the MUIM_Application_RemInputHandler:

```c
static IPTR mDispose(struct IClass *cl,Object *obj,Msg msg)
{
    struct appData          *data = INST_DATA(cl,obj);
    
    ...

   
    DoSuperMethod(cl,obj,MUIM_Application_RemInputHandler,(ULONG)&data->appHandler);
    
    ...

    return DoSuperMethodA(cl,obj,msg);
}             
```
         
## 5.4 Introduction to Sub-processes

In many situations it might happen the need to wait for some resources to become
available or it might also happen the need to interact with some
devices that are slower compared to the execution of one of our own tasks. In
similar conditions, if our program is busy doing a task, it might
hinder the user's capacity to accomplish other tasks - in other words
the program will show a  "busy" state unresponsive to any expected
output that prevents the user from doing further operations, at least until
the perviously started task is finished.

Under normal conditions our program is an unique process in charge to
handle both the user interface input and all the features available to
the user, which can be done sequentially. When the operations involve
CPU use only, through the scheduling techniques available through the
operating system and the new processor's power, those operations might
look almost immediate, with all operations apparently executed in
parallel together, but the reality is different.
If devices or resources are slower than a CPU involved, such as long
disk operations or accessing a printer or so on, the involved task
starts a cycle that contains several wait states, in which the resource
access permissions are checked; of course a similar situation cannot
allow the task to give an adequate possibility of action to the user.
In situations similar to the above the best course of action is to
encapsulate the most heavy procedure and make it run from another
parallel process, so to release the main process from the burden and allow it
to handle other stuff, such as handling user actions. to obtain this
result is possible with the use of the CreateNewProc() call. The
communication between main process and secondary process will be described
according to the IPC standard already described in the paragraphs
above.


### 5.4.1 Introduction to CreateNewProc()

The CreateNewProc() function and its variable parameters version
CreateNewProcTags() allows a running program to generate a new task
that shares the local variables, the current directory and the priority
together with the main task.
This situation is, obviously, configurable through special parameters,
passed through tags to the CreateNewProc() function; furthermore,
together with the traditional Amiga OS 3.x parameters - kept for
compatibility reasons and that might come in handy in some circumstances -
every other incarnation of Amiga OS add several new tags.
Here only the most essential and common tags to create and handle a task
in a portable way through all Amiga OS's will be described: for further
details the reader is invited to look carefully at the CreateNewProc()
documentation of the Amiga-like system of choice.

In order to create a new task at least a standard C function is needed:
that function will impersonate our process and will be passed as a parameter
(NP_Entry) to the CreateNewProc() function.

The standard declaration function to be fed to CreateNewProc() is the
following:

```c
ULONG subproc(STRPTR args, ULONG length)
```

Despite the standard function declaration to be passed in
CreateNewProc() in Amiga OS 4 it will instead look like:

```c
int32 subproc(STRPTR args, int32 length, APTR execbase)
```

and in MorphOS will look like:

```c
void subproc(void) 
```

the first declaration above is supported in all systems for
compatibility reasons; the only thing that concerns the operating system
is the function return; it will need to match one of the AmigaDOS codes
contained in \<dos/dos.h>: RETURN_OK, RETURN_WARN, RETURN_FAIL, RETURN_ERROR.

The CreateNewProc() function relative options are contained as tags
inside the \<dos/dostags.h> header; it is better to view it together
with the dos library autodoc.

Considered that in this case our purpose for a new task is to delegate
operations such as interacting with slow devices, it might be useful
to set it at a lower priority than the main task; to do that, the
NP_Priority tag can be used.

In MorphOS an important CreateNewProc() tag is NP_CodeType, that needs to
be set to the special value CODETYPE_PPC - it will tell the system that
a new PPC native task will be launched.

Another important tag, this time under AmigaOS4, is the NP_Child tag:
if set to TRUE will tell the system that the starting task is a child
of the main task.

Summarising, the code will look like this:

```c
#ifndef __MORPHOS__
#define NP_CodeType TAG_IGNORE
#define CODETYPE_PPC TRUE
#endif

#ifndef __amigaos4__
#define NP_Child TAG_IGNORE
#endif  


ULONG subProcess(void)
{
    ...

    return RETURN_OK;
}

int main(void)
{
    struct Process *pr;
    ...

  
    if (pr = CreateNewProcTags(NP_Entry, subproc,
                                   NP_CodeType,CODETYPE_PPC,//on MorphOS
                                   NP_Child, TRUE,          //on AmigaOS4
                                   NP_Priority,-5, 
                               TAG_DONE))
    {
        ...
    }

    ...

    return 0;
}
```


### 5.4.2 Asynchronous Communication, a practical example

Let's suppose we have a subclass of MUIC_Application and that we need to
launch a sub-task from it. All we really need at this point will be
some data structures inside our subclass data area, obviously a
function to use as a subprocess - called subProcess() and at last a
method to use to initialize all the datas and call CreateNewProcTags(), 
in our example the method MUIM_MyClass_LaunchSubProc.
The sub-task needs to do its own operations in a way that does not slow
down the main program operations; therefore we will need to communicate
asynchronously with the main program, using for the purpose:

- a private communication port for any incoming answers that the main
  program might receive from the sub-task (myPort); 
- the system's port assigned to the sub-task as mailbox where the main
  task will send its messages for the sub-task;
- a structure to be used as IPC message between our main task and the
  sub-task (Mymsg);
- the MUIM_Application_AddInputHandler and MUIM_Application_RemInputHandler 
  methods already explained above (5.3.5);
- a support method for the answers from the main task, in example a
  MUIM_MyClass_FreeResources;

The communication process is identified from the value of a status
variable inside the IPC message, and essentially acts as follows:

1) the main process launches the subprocess and asks to execute what it is
   supposed to (status = PLEASE_EXECUTE_YOUR_WORK);
2) the sub-process does its job, but requests something from the main process
   (status = MAINPROC_I_WANT_SOMETHING);
3) the main process provides an adequate answer to the sub-process  (status 
   HERE_I_GIVE_YOU_WHAT_YOU_WANT);
4) the sub-process finishes the last part of the job and warns the main process
   that it has accomplished all of the job queue (status = QUIT);
5) the main process is finally free to remove from memory all
   unnecessary resources;

It is important to note that the end of a sub-task always includes a
Forbid() call before the canonic ReplyMsg() call. This Forbid() call is
required to tell the system to not execute a context change before the
sub-task finishes completely its job - in this case the ending code.
Why do we need a call like this? Well, let's rethink about the way we created
our sub-process that, for its own nature, is a kind of main process extension
and shares with it some essential data, like the libraries base and
something else. The main process might close before one of the sub-process and
leave it without an essential chunk of required data, that is the
reason for the Forbid() call: it allows the system to close the sub-process
before the main process.
Under Amiga OS 4 the Forbid() call is superfluous in case the NP_Child
tag is set to TRUE: in this case there will be the system itself to take care
not to close the main process before our created sub-process.

The code is the following:

```c
/*****************************************************************************/
#ifndef MEMF_SHARED
#define MEMF_SHARED MEMF_PUBLIC
#endif

enum {   
    PLEASE_EXECUTE_YOUR_WORK=1,
    MAINPROC_I_WANT_SOMETHING,
    HERE_I_GIVE_YOU_WHAT_YOU_WANT,
    QUIT
};  


struct myData
{
    struct MsgPort              *myPort;
    struct Process              *mySubProc;
    struct MUI_InputHandlerNode myHandler;

    ...
};


struct Mymsg
{
  struct Message  msg;

  ...

  LONG            status;
};
/*****************************************************************************/

/*****************************************************************************/
///subProcess()
int subProcess(void)
{
    LONG  wrap;
    STRPTR font;
    struct Process *pr = (struct Process *) FindTask(NULL);
    struct printmsg *mex = NULL;


    #ifdef __MORPHOS__
    struct ExecBase *SysBase = *(struct ExecBase **) 4;
    #endif

    WaitPort (&pr->pr_MsgPort);

    while (!mex)
        mex =(struct printmsg *) GetMsg(&pr->pr_MsgPort);

    if (mex->status == PLEASE_EXECUTE_YOUR_WORK)
    {
      ...

      mex->status = MAINPROC_I_WANT_SOMETHING;
      ReplyMsg ((struct Message *)mex);

      mex = NULL;
      while ( !(mex = (struct Mymsg *)GetMsg (&pr->pr_MsgPort)) )
          WaitPort (&pr->pr_MsgPort);     //wait for reaction from main task
    }
    else if (mex->status == HERE_I_GIVE_YOU_WHAT_YOU_WANT)
    {
      ...
    }

    ...

    mex->status = QUIT;
    #ifndef __amigaos4__
    Forbid();
    #endif
    ReplyMsg ((struct Message *)mex);


    return RETURN_OK;
}
///
/*****************************************************************************/

/*****************************************************************************/
///MUIM_MyClass_LaunchSubProc
static IPTR LaunchSubProcMethod(struct IClass *cl,Object *obj, Msg msg)
{
    struct myData    *data = INST_DATA(cl,obj);
    struct Mymsg   *mex;

    if (data->myPort)
    {
        MUI_RequestA(obj,NULL,0,
                        (char *)"Requester",
                        (char *)"Ok",
                        (char *)"Just in running...",
                     NULL);

        return (IPTR) FALSE;
    }

    data->mySubProc=NULL;

    if ((mex = AllocVec (sizeof(struct Mymsg),MEMF_CLEAR|MEMF_SHARED)))
    {
        if ((data->myPort = CreateMsgPort()))
        {
            if((data->mySubProc = CreateNewProcTags(NP_Entry, (ULONG) subProc,
                                                      NP_CodeType,CODETYPE_PPC,//on MorphOS
                                                      NP_Child, TRUE,          //on AmigaOS4
                                                      NP_Priority,-5,
                                                    TAG_DONE)))
            {
                data->myHandler.ihn_Object   = obj;
                data->myHandler.ihn_Signals  = 1L<<data->myPort->mp_SigBit;
                data->myHandler.ihn_Method   = MUIM_App_FreeResources;
                data->myHandler.ihn_Flags    = 0;
               
                DoSuperMethod(cl,obj,MUIM_Application_AddInputHandler,(IPTR)&data->myHandler);

                ...

                mex->status = PLEASE_EXECUTE_YOUR_WORK;
                mex->msg.mn_ReplyPort = data->myPort;
                
                /* establish communication with sub task */
                PutMsg (&data->printSubProcess->pr_MsgPort,(struct Message *)mex);   
            }
        }
    }


    return (IPTR) TRUE;
}
///
/*****************************************************************************/

/*****************************************************************************/
///MUIM_MyClass_FreeResources
static IPTR FreeResourcesMethod(struct IClass *cl,Object *obj,Msg msg)
{
    struct myData *data = INST_DATA(cl,obj);
    IPTR ret = FALSE;

    if (data->myPort && data->mySubProcess)
    {
        struct Mymsg *mex = NULL;

        /* wait for reply from sub proc */
        while((mex = (struct Mymsg  *)GetMsg(data->myPort)) != NULL)
        {

            ...

            if (status == MAINPROC_I_WANT_SOMETHING)
            {
              ...

              mex->status = HERE_I_GIVE_YOU_WHAT_YOU_WANT;
              mex->msg.mn_ReplyPort = data->myPort;

              /* reply to sub proc*/
              PutMsg (&data->mySubProcess->pr_MsgPort,(struct Message *)mex);

              ret = TRUE;
              break;
            }
            else if (mex->status==QUIT)
            {
               
                DoSuperMethod(cl,obj,MUIM_Application_RemInputHandler,(IPTR)&data->myHandler);

                DeleteMsgPort(data->myPort);

                data->myPort=NULL;


                FreeVec(mex);

                ret = TRUE;
                break;
            }
        }
    }

    return ret;
}
///
/****************************************************************************/
```
 