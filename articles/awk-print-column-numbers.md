# Print Column Numbers for Any Command Output

Generic awk one-liner to identify column positions — useful when building filters and you're not sure which `$N` holds the value you need.

## The Pattern

```bash
<command> | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}'
```

This prints each field prefixed with its column number, making it easy to identify which `$N` to use in subsequent awk filters.

## Examples

### iotop

```bash
sudo iotop -bot -n 1 -P -qqq | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}'
```

```text
1=03:15:47 2=112498 3=be/4 4=dcornesc 5=0.00 6=B/s 7=33.29 8=M/s 9=?unavailable? 10=dd 11=if=/dev/zero ...
```

### ps

```bash
ps aux | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}' | head -5
```

### df

```bash
df -h | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}'
```

### ss / netstat

```bash
ss -tuln | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}'
```

### free

```bash
free -h | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}'
```

### top (batch mode)

```bash
top -bn1 | awk '{for(i=1;i<=NF;i++) printf "%d=%s ", i, $i; print ""}' | head -15
```

## Then Use It

Once you know the column numbers, build your filter:

```bash
# Example: only show iotop processes with write > 1 MB/s (column 7)
sudo iotop -bot -n 1 -P -qqq | awk '/M\/s/ && ($7 > 1) {print}'

# Example: show processes using more than 5% CPU (column 3 in ps aux)
ps aux | awk '$3 > 5.0 {print}'

# Example: show filesystems over 80% usage (column 5 in df -h)
df -h | awk '{gsub(/%/,"",$5)} $5 > 80 {print}'
```
