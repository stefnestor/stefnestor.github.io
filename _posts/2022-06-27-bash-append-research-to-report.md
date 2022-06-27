
Bash append Research to Reports

**CODE**:
```
### use
# execute command, then run: append "!!"
function append {
  if [[ -z $NOTE ]]; then
    echo "🦖 !saved; set NOTE & repeat"
  else
    input="$*"
    # input='ct cat_indices.txt | grep .kibana_task_manager_1'
    FILEa=$HOME_NOTE/$NOTE'.md'

    echo >> $FILEa
    echo "\`\`\`" >> $FILEa
    echo "$ #" $(date +%Y%m%d-%H%M%S) >> $FILEa
    echo "$ "$input >> $FILEa
    echo "$(eval $input)" >> $FILEa
    echo "\`\`\`" >> $FILEa
  fi
}
```

**RUN**:

```
Last login: Mon Jun 27 11:59:23 on ttys000
$ NOTE="TMP"
$ echo "stef"
stef
$ append "!!"
append "echo "stef""
$
```

**RESULT**: 
`TMP.md` (auto-created and) added
`````````
```
$ # 20220627-120212
$ echo stef
stef
```
`````````