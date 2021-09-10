---
layout: post
title: "Awesome selenium video recording with ffmpeg"
date: 2021-09-10
categories: testing selenium video
---

**TLDR**: [Script On Github](https://gist.github.com/joshzcold/d40e7ee47749851b6dda32314b5efab9)

If your not using a paid service that runs your testing you probably are trying to roll your own test recording.

This method that attaches to the background xserver generated when using `xfvb` for background browser testing is a pretty good solution.

It will give a side-by-side of the browser interaction as well as your testing console being outputted while testing.

<p align="center">
<img src="{{site.baseurl}}/assets/img/testVideo.png"
style="width: 1000px;"
>
</p>

## Script Sections

We need to start our browser testing in the background and then keep track of the display

**Find an available display on the server**

This could be a useful script snippet for any `Xvfb` operations

```sh
local tmp=$(mktemp)

# -displayfd finds unused display and sends it to file descriptor 1
Xvfb -displayfd 1 -screen 0 1920x1080x16 1>$tmp 2>/dev/null &

# wait until Xvfb finds a display with output in tmp file
# 5 seconds to find a display so we dont infini-loop
kill_count=0
while ! grep -qP "\d" $tmp;do
  [ "$kill_count" -gt 30 ] && echo "Xvfb couldn't get a display within limit" && exit 1
  sleep 1
  ((kill_count=kill_count+1))
  echo xvfb: $tmp $kill_count
done
xvfb_display=$(cat $tmp)

```

**Start video recording in background attached to Xvfb display**

```sh
function startVideo(){
  # in the background wait until the testing command
  # starts a log so we can attach to it in ffmpeg
  kill_count=0
  while [ ! -f $video_output/$1.log ];do
    [ "$kill_count" -gt 30 ] && echo "could not get testing log within limit" && exit 1
    sleep 1
    ((kill_count=kill_count+1))
    echo $video_output/$1.log $kill_count
  done

  # start recording video in background
  if [ -f $video_output/$1.log ];then
    # ffmpeg can take new commands via file content when tailed
    echo > $video_cmds
    set -x
      tail -f $video_cmds | ffmpeg -f x11grab -y \
      -video_size 1920x1080 -hide_banner \
      -loglevel error -i :$xvfb_display \
      -codec:v libx264 -r 12 \
        -vf "drawtext=textfile=$video_output/$1.log \
        :fontcolor=white \
        :fontsize=18 \
        :box=1 \
        :boxcolor=black@0.7 \
        :reload=1
        :x=w-tw-20:y=h-th-20" \
      $video_output/$1$video_extension
    set +x
  else
    echo "no log file for ffmpeg video recording. skipping video recording"
  fi
}

startVideo $test_suite_name &
```

Lets go over some details

`-vf "drawtext=textfile=$video_output/$1.log \` insert text into the video. Normally this is done for a title screen or a persistent header during the video. This doesn't work for a scrolling log file unless we include `reload=1`

`:x=w-tw-20:y=h-th-20" \` will move the browser recording to the second half of the screen. In your testing code you will want to start your browser with 1/2 or 3/4 the size of `1920x1080`

`-i $xvfb_display` attaches to our available server

**Start testing under display in background**

```sh
function testCommand(){

  # 1. Run gradle with arguments
  # 2. Tee stdout to tty and to stdin for sed
  # 3. replace lines >60 with a newline
  echo "Loading..." > $video_output/$2.log
  set -x
  ./gradlew test $gradle_options --rerun-tasks $1$2 2>&1 |
    tee >(sed -u -e "s/.\{75\}/&\n/g" >> $video_output/$2.log)
  set +x
}

# start testing in background
DISPLAY=":$xvfb_display" testCommand &
```

**Cleanup**

These commands will clean up our background jobs if any of the jobs fail or you manually trigger a SIG-INT with <kbd>ctrl-c</kbd>

```sh
# Ctrl + C kill background jobs
trap 'pkill -P $$' TERM INT HUP
```

If we get the exit code of our testing command we can determine if we want to delete the video on passing tests.

The command `wait` waits for the background pid to finish and returns the pid exit code

`exit $test_fail_exit` is a normal exit that will stop other background jobs. `$test_fail_exit` is a variable that will determine if we want a failing exit code on test failure

```sh
DISPLAY=":$xvfb_display" testCommand $1 $2 &
test_pid=$!

# wait and output exit of background process
if wait $test_pid; then
  echo q >> $video_cmds # send ffmpeg exit command
  sleep 2
  if ! $keep_logs;then
    rm -f $video_output/$2.log
  fi
  echo
  echo "Tests succeed, delete video -> $video_output/$2$video_extension"
  rm -f $video_output/$2$video_extension
  exit 0
else
  echo
  echo "1 or more test failed, keeping -> $video_output/$2$video_extension"
  echo q >> $video_cmds # send ffmpeg exit command
  sleep 2
  if ! $keep_logs;then
    rm -f $video_output/$2.log
  fi
  # kill everything else
  exit $test_fail_exit
fi

```

## Full Script

```sh
#!/usr/bin/env bash
# set -x
# PS4='+${LINENO}: '
set -euo pipefail
IFS=$'\n\t'

method=""
class=""
package=""
directory=""
test_fail_exit=0
video=false
keep_logs=false
video_output=$PWD
testIndicator="@Test"
video_extension=".mp4"
xvfb_display=""
video_cmds=$(mktemp)

function startVideo(){
  kill_count=0
  while [ ! -f $video_output/$1.log ];do
    [ "$kill_count" -gt 30 ] && echo "could not get testing log within limit" && exit 1
    sleep 1
    ((kill_count=kill_count+1))
    echo $video_output/$1.log $kill_count
  done

  if [ -f $video_output/$1.log ];then
    # start recording video in background
    echo > $video_cmds
    set -x
      tail -f $video_cmds | ffmpeg -f x11grab -y \
      -video_size 1920x1080 -hide_banner \
      -loglevel error -i :$xvfb_display \
      -codec:v libx264 -r 12 \
        -vf "drawtext=textfile=$video_output/$1.log \
        :fontcolor=white \
        :fontsize=18 \
        :box=1 \
        :boxcolor=black@0.7 \
        :reload=1
        :x=w-tw-20:y=h-th-20" \
      $video_output/$1$video_extension
    set +x
  else
    echo "no log file for ffmpeg video recording. skipping video recording"
  fi
}

function testCommand(){

  # 1. Run gradle with arguments
  # 2. Tee stdout to tty and to stdin for sed
  # 3. replace lines >60 with a newline
  echo "Loading..." > $video_output/$2.log
  if $video; then
    set -x
    ./gradlew test $gradle_options --rerun-tasks $1$2 2>&1 |
      tee >(sed -u -e "s/.\{75\}/&\n/g" >> $video_output/$2.log)
    set +x
  else
    set -x
    ./gradlew test $gradle_options --rerun-tasks $1$2
    set +x
  fi
}

function testDirectory(){
  for f in $(find $directory -name "*.groovy")
  do
      package=$(head -1 $f | awk '{print $2}')
      #xargs removes white space
      class=$(grep -Po "(?<=class)\s\w+\s" $f | xargs)
      execute="${package}.${class}"

      if $video; then
        withVideo "--tests=" "$execute"
      else
        testCommand "--tests=" "$execute"
      fi

  done
}

function withVideo(){
  local tmp=$(mktemp)

  # Ctrl + C kill background jobs
  trap 'pkill -P $$' TERM INT HUP

  # -displayfd finds unused display and send it to file descriptor 1
  Xvfb -displayfd 1 -screen 0 1920x1080x16 1>$tmp 2>/dev/null &

  # wait until Xvfb finds a display with output in tmp file
  # 5 seconds to find a display so we dont infini-loop
  kill_count=0
  while ! grep -qP "\d" $tmp;do
    [ "$kill_count" -gt 30 ] && echo "Xvfb couldn't get a display within limit" && exit 1
    sleep 1
    ((kill_count=kill_count+1))
    echo xvfb: $tmp $kill_count
  done
  xvfb_display=$(cat $tmp)

  mkdir -p $video_output

  startVideo $2 &

  # send the "headed" browser to DISPLAY which is headless
  DISPLAY=":$xvfb_display" testCommand $1 $2 &
  test_pid=$!

  # wait and output exit of background process
  if wait $test_pid; then
    echo q >> $video_cmds # send ffmpeg exit
    sleep 2
    if ! $keep_logs;then
      rm -f $video_output/$2.log
    fi
    echo
    echo "Tests succeed, delete video -> $video_output/$2$video_extension"
    rm -f $video_output/$2$video_extension
    exit 0
  else
    echo
    echo "1 or more test failed, keeping -> $video_output/$2$video_extension"
    echo q >> $video_cmds # send ffmpeg exit
    sleep 2
    if ! $keep_logs;then
      rm -f $video_output/$2.log
    fi
    # kill everything else
    exit $test_fail_exit
  fi
}

function testSuite(){
  if $video; then
    withVideo "-Dsuite=" "$suite"
  else
    testCommand "-Dsuite=" "$suite"
  fi
}

function testPackage(){
  if $video; then
    withVideo "--tests=" "$package"
  else
     testCommand "--tests=" "$package"
  fi
}

function testClass() {
  echo Testing all @Test in class
  file="$(grep -r -w "class ${class}" | cut -d ":" -f 1 || true)"
  # Empty Check
  if [[ -z "$file" ]]; then
   echo "Error: Couldn't find an exact match"
   exit 1
  fi

  if [[ $(echo "${file}" | wc -l)  -gt 1 ]]; then
    echo Error: Found more than one file that has that class. exiting
    exit 1
  else
      package=$(head -1 ${file} | awk '{print $2}')
      execute="${package}.${class}"
      echo going to try and run class: ${execute}
      if $video; then
        withVideo "--tests=" "$execute"
      else
        testCommand "--tests=" "$execute"
      fi
  fi
}

function testMethod() {
  echo Testing a @Test method within a class
  file="$(grep -r "${testIndicator}" -A1 | grep -w "${method}" | grep  -v "${testIndicator}" | tr -s ' ' | cut -d ' ' -f 1 | awk '{print substr($1, 1, length($1)-1)}' || true)"

  # Empty Check
  if [[ -z "$file" ]]; then
   echo "Error: Couldn't find an exact match"
   exit 1
  fi

  if [[ $(echo "${file}" | wc -l)  -gt 1 ]]; then
    echo Error: Found more than one file that has that method. exiting
    exit 1
  else
    class=$(grep "class" ${file} | awk '{print $2}' )
    if [[ $(echo "$class" | wc -l) -gt 1 ]]; then
      echo Error: Found more than one class in file. This script can only work with one class
      exit 1
    else
      package=$(head -1 ${file} | awk '{print $2}')
      execute="${package}.${class}.${method}"
      echo going to try and run method:  ${execute}
      if $video; then
        withVideo "--tests=" "$execute"
      else
        testCommand "--tests=" "$execute"
      fi
    fi
  fi
}

#===  FUNCTION  ================================================================
#         NAME:  usage
#  DESCRIPTION:  Display usage information.
#===============================================================================
function usage ()
{
  echo "
  =====================================================
  Wrapper script for easier commandline test operations

  Finds test classes or methods and resolves the package
  to pass into build tools like maven or gradle
  =====================================================
  Usage :  $0 [options] [gradle arguments]
    Options:
    -h Display this message
    -d Run all tests recursivley found in directory
    -m Search for and run Method Test
    -l Keep logs from tests
    -p Put in full package declaration
    -s Run suite, must be function in build file (build.gradle)
    -x Exit code on test failure
    -v Record video (executes test in background)
    -o Video output directory (will be created)
    -c Search for and run tests inside of Class

   Examples:
    ./test.sh -p com.microfocus.sspr.tests.smokeTests.canModule.CanLookup
    ./test.sh -c CanLookup -DinternalSysProp=something
    ./test.sh -m methodInCanLookup

    # exit with exit code 1 on failure
    ./test.sh -x 1

    # Record video to this directory
    ./test.sh -m -v -o /home/me/videos methodInCanLookup"


}    # ----------  end of function usage  ----------


#-----------------------------------------------------------------------
#  Handle command line arguments
#-----------------------------------------------------------------------

while getopts "hlvd:s:x:m:c:p:o:" opt;
do
  case $opt in
  h)
    usage; exit 0   ;;
  v)
    video=true;;
  l)
    keep_logs=true;;
  o)
    video_output=$OPTARG;;
  x)
    test_fail_exit=$OPTARG;;
  m)
    method=$OPTARG
    shift $(($OPTIND - 1))
    gradle_options="$@"
    testMethod; exit 0   ;;
  c)
    class=$OPTARG
    shift $(($OPTIND - 1))
    gradle_options="$@"
    testClass; exit 0   ;;
  p)
    package=$OPTARG
    shift $(($OPTIND - 1))
    gradle_options="$@"
    testPackage; exit 0   ;;
  s)
    suite=$OPTARG
    shift $(($OPTIND - 1))
    gradle_options="$@"
    testSuite; exit 0   ;;
  d)
    directory=$OPTARG
    shift $(($OPTIND - 1))
    gradle_options="$@"
    testDirectory; exit 0   ;;
  *)  echo -e "\n  Option does not exist : OPTARG\n"
        usage; exit 1   ;;

  esac    # --- end of case ---
done
```
