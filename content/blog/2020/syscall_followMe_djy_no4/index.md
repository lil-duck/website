---
title: "系统调用跟我学(4)"
date: 2020-06-23T17:31:21+08:00
author: "作者：雷镇 编辑：戴君毅"
keywords: ["系统调用"]
categories : ["系统调用"]
banner : "img/blogimg/syscall.png"
summary : "这是本专栏中进程相关的系统调用的最后一篇，用2个实例演示了以往学习的内容。其一是Mini Shell，仿常用的Bash而做，但对其作了大大简化；其二是一个Daemon程序，可以使读者一窥服务器编程的端倪。"
---

> 这是本专栏中进程相关的系统调用的最后一篇，用2个实例演示了以往学习的内容。其一是Mini Shell，仿常用的Bash而做，但对其作了大大简化；其二是一个Daemon程序，可以使读者一窥服务器编程的端倪。

**1.13 Shell**

对Linux不是太陌生的读者都应该对Shell有一定的了解，就是这个程序在我们登陆后自动执行，打印出一个$符号，然后等待我们输入命令。Linux下最常用的Shell应用程序是Bash，绝大部分Linux发行版默认安装的都是它。下面我们也来亲手编写一个Shell程序，这个Shell远远不如Bash复杂，但也能满足我们一般的使用，下面，我们就开始。

首先，给这个Shell取一个名字，不妨就叫做Mini Shell。

Linux系统的命令分为内部命令和外部命令两种，内部命令由Shell程序实现，如cd、echo等，Linux的内部命令数量有限，而且绝大部分都很少用到。而每一个Linux外部命令都是一个单独的应用程序，我们非常熟悉的ls、cp等绝大多数命令都是外部命令，这些命令都以可执行文件的形式存在，绝大部分放在目录/bin和/sbin中。这样一来，我们编程的难度就可以大大下降了，我们只需要实现很有限的内部命令，对于其它的输入，统统当作应用程序来执行即可。

为了简单明了起见，Mini Shell只实现了2个内部命令： 
1、cd	用于切换目录，和我们熟悉的命令cd类似，除了没有那么多的附加功能。 
2、quit	用于退出Mini Shell。

下面是程序清单：

```
1: 	/* mshell.c */
2: 	#include <sys/types.h>
3: 	#include <unistd.h>
4: 	#include <sys/wait.h>
5: 	#include <string.h>
6: 	#include <errno.h>
7: 	#include <stdio.h>
8: 
9: 	void do_cd(char *argv[]);
10: void execute_new(char *argv[]);
11:
12:	main()
13:	{
14:		char *cmd=(void *)malloc(256*sizeof(char));
15:		char *cmd_arg[10];
16:		int cmdlen,i,j,tag;
17:	
18:		do{
19:			/* 初始化cmd */
20:			for(i=0;i<255;i++) cmd[i]='\0';
21:	
22:			printf("-=Mini Shell=-*| ");
23:			fgets(cmd,256,stdin);
24:	
25:			cmdlen=strlen(cmd);
26:			cmdlen--;
27:			cmd[cmdlen]='\0';
28:	
29：			/* 把命令行分解为指针数组cmd_arg */
30：			for(i=0;i<10;i++) cmd_arg[i]=NULL;
31：			i=0; j=0; tag=0;
32：			while(i<cmdlen && j<10){
33：				if(cmd[i]==' '){
34：					cmd[i]='\0';
35：					tag=0;
36：				}else{
37：					if(tag==0)
38：						cmd_arg[j++]=cmd+i;
39：					tag=1;
40：				}
41：				i++;
42：			}
43：			
44：			/* 如果参数超过10个，就打印错误，并忽略当前输入 */
45：			if(j>=10 && i<cmdlen){
46：				printf("TOO MANY ARGUMENTS\n");
47：				continue;
48：			}
49：			
50：			/* 命令quit：退出Mini Shell */
51：			if(strcmp(cmd_arg[0],"quit")==0)
52：				break;
53：	
54：			/* 命令cd */
55：			if(strcmp(cmd_arg[0],"cd")==0){
56：				do_cd(cmd_arg);
57：				continue;
58：			}
59：			
60：			/* 外部命令或应用程序 */
61：			execute_new(cmd_arg);
62：		}while(1);
63：	}
64：	
65：	/* 实现cd的功能 */
66：	void do_cd(char *argv[])
67：	{
68：		if(argv[1]!=NULL){
69：			if(chdir(argv[1])<0)
70：				switch(errno){
71：				case ENOENT:
72：					printf("DIRECTORY NOT FOUND\n");
73：					break;
74：				case ENOTDIR:
75：					printf("NOT A DIRECTORY NAME\n");
76：					break;
77：				case EACCES:
78：					printf("YOU DO NOT HAVE RIGHT TO ACCESS\n");
79：					break;
80：				default:
81：					printf("SOME ERROR HAPPENED IN CHDIR\n");
82：				}
83：		}
84：	
85：	}
86：	
87：	/* 执行外部命令或应用程序 */
88：	void execute_new(char *argv[])
89：	{
90：		pid_t pid;
91：	
92：		pid=fork();
93：		if(pid<0){
94：			printf("SOME ERROR HAPPENED IN FORK\n");
95：			exit(2);
96：		}else if(pid==0){
97：			if(execvp(argv[0],argv)<0)
98：				switch(errno){
99：				case ENOENT:
100：					printf("COMMAND OR FILENAME NOT FOUND\n");
101：					break;
102：			case EACCES:
103：					printf("YOU DO NOT HAVE RIGHT TO ACCESS\n");
104：					break;
105：	        default:
106：	                printf("SOME ERROR HAPPENED IN EXEC\n");
107：				}
108：		exit(3);
109：	}else 
110：		wait(NULL);
111：}
```



这个程序稍稍有点长，我们来对它作一下详细的解释：

函数main：

14行：定义字符串cmd，用于接收用户输入的命令行。 
15行：定义指针数组cmd_arg，它的形式和作用都与我们熟悉的char *argv[]一样。

从以上2个定义可以看出Mini Shell对命令输入的2个限制：首先，用户输入的命令行必须在255个字符之内（除去字符串结束标志'\0'）；其次，命令行的参数个数不得超过10个（包括命令本身在内）。

18行：进入一个do-while循环，这个循环是本程序的主体部分，基本思想是"等待输入命令--处理已输入命令--等待输入命令"。

22行：打印输入提示信息。在Mini Shell中，你可以随意定自己喜欢的命令输入提示信息，本程序中使用了"-=Mini Shell=-*| "，是不是有点像一个CS高手？如果不喜欢，你可以用任意的字符替换它。

23行：接收用户输入。

25-27行：fgets接受输入时，会将输入字符串时末尾的换行符（"\n"）一起接受，这是我们不需要的，所以要把它去掉。本程序中简单的用字符串结束标志'\0'覆盖了字符串cmd的最后一个字符来实现这个目的。

30行：初始化指针数组cmd_arg。

32-42行：对输入进行分析，将cmd中参数间的空格用'\0'填充，并把各参数的起始地址分别赋与cmd_arg数组。这样就把cmd分解成了cmd_arg，但分解后的各命令参数仍然使用着cmd的内存空间，所以在命令执行结束前不宜对cmd另外赋值。

45行：如果还未分析到输入字符串的末尾（i<cmdlen），而分析出的参数已经达到或超过了10个（j>=10），就认为输入的命令行超出了10个参数的限制，打印错误并重新接收命令。

51-52行：内部命令quit：字符串cmd_arg[0]就是命令本身，如果命令是quit，则退出循环，也就等于退出该程序。

55-58行：内部命令cd：调用函数do_cd()完成cd命令的动作。

61行：对于其它的外部命令和应用程序，调用函数execute_new()执行。

函数do_cd：

68行：仅仅考虑紧跟在命令后面的参数argv[1]，而不再考虑其它的参数。如果这个参数存在，就把它作为要转换的目录。

69行：调用系统调用chdir切换当前目录，参见附录1。

70-82行：对chdir可能出现的错误进行处理。

函数execute_new：

92行：调用系统调用fork产生新的子进程。

93行：如果返回负值，说明fork调用出现错误。

96行：如果返回0，说明当前进程是子进程。

97行：调用execvp执行新的应用程序，并检测调用是否出错（返回负值）。这里使用execvp的原因是它可以自动在各默认目录里寻找目标应用程序的位置，而不必我们自己编程实现。

98-107行：对execvp可能出现的错误进程处理。

108行：如果execvp的执行出现错误，子进程在这里终止。表面上看起来，这个exit是接着97行的错误判断的下一行语句，而非if语句的一部分，似乎无论调用execvp成功与否都会接着执行exit。但事实上，如果execvp调用成功的话，这个进程将会被新的程序代码填充，因而根本不可能执行到这一行。反之，如果执行到了这一行，说明前面的execvp调用一定出现了错误。这样的效果和exit被包含在if语句中的效果是完全一样的。

109行：如果fork返回其它值，说明当前进程是父进程。

110行：调用系统调用wait。wait在这里有两个作用：

1. 使父进程在此暂停，等待子进程执行完毕。这样，就可以等子进程的所有信息全部输出完毕后才打印命令提示符，等待下一条命令的输入，从而避免了命令提示符和应用程序输出混杂在一起的现象。
2. 收集子进程退出后留下的僵尸进程。可能有读者一直对这个问题存有疑问--"我们编程生成的子进程由我们自己设计的父进程负责收集，但我们手动执行的这个父进程由谁收集呢？"现在大家应该明白了，我们从命令行执行的所有进程最后都是由shell收集的。

关于Mini Shell的编译和运行，这里就不再敷述了，有兴趣的读者可以自行动手实验，或者对这个程序进行改进，使之更接近甚至超过我们正使用的Bash。



---



**1.14 daemon进程**

**1.14.1 了解daemon进程**

这又是一个有趣的概念，daemon在英语中是"精灵"的意思，就像我们经常在迪斯尼动画里见到的那些，有些会飞，有些不会，经常围着动画片的主人公转来转去，啰里啰唆地提一些忠告，时不时倒霉地撞在柱子上，有时候还会想出一些小小的花招，把主人公从敌人手中救出来，正因如此，daemon有时也被译作"守护神"。所以，daemon进程在国内也有两种译法，有些人译作"精灵进程"，有些人译作"守护进程"，这两种称呼的出现频率都很高。

与真正的daemon相似，daemon进程也习惯于把自己隐藏在人们的视线之外，默默为系统做出贡献，有时人们也把它们称作"后台服务进程"。daemon进程的寿命很长，一般来说，从它们一被执行开始，直到整个系统关闭，它们才会退出。几乎所有的服务器程序，包括我们熟知的Apache和wu-FTP，都用daemon进程的形式实现。很多Linux下常见的命令如inetd和ftpd，末尾的字母d就是指daemon。

为什么一定要使用daemon进程呢？Linux中每一个系统与用户进行交流的界面称为终端（terminal），每一个从此终端开始运行的进程都会依附于这个终端，这个终端就称为这些进程的控制终端（Controlling terminal），当控制终端被关闭时，相应的进程都会被自动关闭。关于这点，读者可以用X-Window中的XTerm试验一下，（每一个XTerm就是一个打开的终端，）我们可以通过键入命令启动应用程序，比如：

**$netscape**

然后我们关闭XTerm窗口，刚刚启动的netscape窗口也会随之一同突然蒸发。但是daemon进程却能够突破这种限制，即使对应的终端关闭，它也能在系统中长久地存在下去，如果我们想让某个进程长命百岁，不因为用户或终端或其他的变化而受到影响，就必须把这个进程变成一个daemon进程。

**1.14.2 daemon进程的编程规则**

如果想把自己的进程变成daemon进程，我们必须严格按照以下步骤进行：

1. 调用fork产生一个子进程，同时父进程退出。我们所有后续工作都在子进程中完成。这样做我们可以：

   1. 如果我们是从命令行执行的该程序，这可以造成程序执行完毕的假象，shell会回去等待下一条命令；
   2. 刚刚通过fork产生的新进程一定不会是一个进程组的组长，这为第2步的执行提供了前提保障。

   这样做还会出现一种很有趣的现象：由于父进程已经先于子进程退出，会造成子进程没有父进程，变成一个孤儿进程（orphan）。每当系统发现一个孤儿进程，就会自动由1号进程收养它，这样，原先的子进程就会变成1号进程的子进程。

    

2. 调用setsid系统调用。这是整个过程中最重要的一步。setsid的介绍见附录2，它的作用是创建一个新的会话（session），并自任该会话的组长（session leader）。如果调用进程是一个进程组的组长，调用就会失败，但这已经在第1步得到了保证。调用setsid有3个作用：

   1. 让进程摆脱原会话的控制；
   2. 让进程摆脱原进程组的控制；
   3. 让进程摆脱原控制终端的控制；

   总之，就是让调用进程完全独立出来，脱离所有其他进程的控制。

    

3. 把当前工作目录切换到根目录。如果我们是在一个临时加载的文件系统上执行这个进程的，比如：/mnt/floppy/，该进程的当前工作目录就会是/mnt/floppy/。在整个进程运行期间该文件系统都无法被卸下（umount），而无论我们是否在使用这个文件系统，这会给我们带来很多不便。解决的方法是使用chdir系统调用把当前工作目录变为根目录，应该不会有人想把根目录卸下吧。关于chdir的用法，参见附录1。 
   当然，在这一步里，如果有特殊的需要，我们也可以把当前工作目录换成其他的路径，比如/tmp。 

4. 将文件权限掩码设为0。这需要调用系统调用umask，参见附录3。每个进程都会从父进程那里继承一个文件权限掩码，当创建新文件时，这个掩码被用于设定文件的默认访问权限，屏蔽掉某些权限，如一般用户的写权限。当另一个进程用exec调用我们编写的daemon程序时，由于我们不知道那个进程的文件权限掩码是什么，这样在我们创建新文件时，就会带来一些麻烦。所以，我们应该重新设置文件权限掩码，我们可以设成任何我们想要的值，但一般情况下，大家都把它设为0，这样，它就不会屏蔽用户的任何操作。 
   如果你的应用程序根本就不涉及创建新文件或是文件访问权限的设定，你也完全可以把文件权限掩码一脚踢开，跳过这一步。 

5. 关闭所有不需要的文件。同文件权限掩码一样，我们的新进程会从父进程那里继承一些已经打开了的文件。这些被打开的文件可能永远不被我们的daemon进程读或写，但它们一样消耗系统资源，而且可能导致所在的文件系统无法卸下。需要指出的是，文件描述符为0、1和2的三个文件（文件描述符的概念将在下一章介绍），也就是我们常说的输入、输出和报错这三个文件也需要被关闭。很可能不少读者会对此感到奇怪，难道我们不需要输入输出吗？但事实是，在上面的第2步后，我们的daemon进程已经与所属的控制终端失去了联系，我们从终端输入的字符不可能达到daemon进程，daemon进程用常规的方法（如printf）输出的字符也不可能在我们的终端上显示出来。所以这三个文件已经失去了存在的价值，也应该被关闭。 

下面，就然我们亲眼看一个daemon进程的诞生：

**1.14.3 一个daemon程序**

```
/* daemon.c */
#include<unistd.h>
#include<sys/types.h>
#include <sys/stat.h>
#define MAXFILE 65535
main()
{
	pid_t pid;
	int i;
        pid=fork();
	if(pid<0){
		printf("error in fork\n");
		exit(1);
	}else if(pid>0) 
		/* 父进程退出 */
		exit(0); 
	/* 调用setsid */
        setsid();
	/* 切换当前目录 */
        chdir("/");
	/* 设置文件权限掩码 */
	umask(0);
	/* 关闭所有可能打开的不需要的文件 */
	for(i=0;i<MAXFILE;i++)
		close(i);
	/* 
	   到现在为止，进程已经成为一个完全的daemon进程，
	   你可以在这里添加任何你要daemon做的事情，如：
	*/ 
	for(;;)
		sleep(10);
}
```

编译和运行的任务就交给读者们自己完成。daemon进程不像其他进程一样有很抢眼的运行结果，基本上它只是毫不声张地做自己的事。你不可能看到任何东西，但可以用"ps -ajx"命令观察一下你的daemon进程的状态和一些参数。



---



**1.15 附录**

**1.15.1 系统调用chdir**

```
  #include <unistd.h>
  int chdir(const char *path);
```

chdir的作用是改变当前工作目录。进程的当前工作目录一般是应用程序启动时的目录，一旦进程开始运行后，当前工作目录就会保持不变，除非调用chdir。chdir只有1个字符串参数，就是要转去的路径。例如：

```
 chdir("/"); 
```

进程的当前路径就会变为根目录。

**1.15.2 系统调用setsid**

```
 #include <unistd.h>
 pid_t setsid(void);
```

一个会话（session）开始于用户登陆，终止于用户退出，在此期间该用户运行的所有进程都属于这个会话，除非进程调用setsid系统调用。

系统调用setsid不带任何参数，调用之后，调用进程就会成立一个新的会话，并自任该会话的组长。

**1.15.3 系统调用umask**

```
 #include <sys/types.h>
 #include <sys/stat.h>
 mode_t umask(mode_t mask);
```

系统调用umask可以设定一个文件权限掩码，用户可以用它来屏蔽某些权限，以防止误操作导致给予某些用户过高的权限。



**参考资料**

- Linux man pages

- Advanced Programming in the UNIX Environment by W. Richard Stevens, 1993



**关于作者**

雷镇，您可以通过电子邮件 [leicool@21cn.com](mailto:leicool@21cn.com?cc=)和他联系。