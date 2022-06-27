{:toc}

### code
```
### use
# execute command, then run: append "!!"
function append {
  if [[ -z $NOTE ]]; then
    echo "👻 not saved; set NOTE & repeat"
  else
    input="$*"
    FILEa='/Users/stef/downloads'/$NOTE'.md'

    echo >> $FILEa
    echo "\`\`\`" >> $FILEa
    echo "$ #" $(date +%Y%m%d-%H%M%S) >> $FILEa
    echo "$ "$input >> $FILEa
    echo "$(eval $input)" >> $FILEa
    echo "\`\`\`" >> $FILEa
  fi
}
```

### run

```
Last login: Mon Jun 27 11:59:23 on ttys000
$ NOTE="TMP"
$ echo "stef"
stef
$ append "!!"
append "echo "stef""
$
```

### result

`TMP.md` (auto-created and) added
`````````
```
$ # 20220627-120212
$ echo stef
stef
```
`````````