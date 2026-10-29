grep -h DriverTaskQueueService /logs/ingestion/ingestion.log | head -20 > /tmp/sample.log

#!/usr/bin/env bash
# Usage: ./ingestion_stats.sh [logfile] [YYYY-MM-DD]
# Needs: clush, gawk. Output: CSV on stdout.
LOG="${1:-/logs/ingestion/ingestion.log}"
DAY="${2:-}"
PATTERN="${DAY:+^$DAY.*}DriverTaskQueueService"

clush -a -N "grep -h -E '$PATTERN' $LOG" 2>/dev/null |
gawk '
{
  gsub(/\r/, "")
  gsub(/\([+-][0-9]+\)/, "")      # drop (+12) / (-4) deltas
  if (match($0, /([0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9] [0-9:]+),[0-9]+ .*DriverTaskQueueService - \[([^#]+)#([^]]+)\] (.*)$/, m))
    print m[1] "|" m[2] "|" m[3] "|" m[4]
}' |
LC_ALL=C sort -t'|' -k3,3 -k1,1 |
gawk -F'|' '
function secs(s,  t,n,u) {          # 1M30S -> 90, "-" -> 0
  t = 0
  while (match(s, /^[0-9]+[DHMS]/)) {
    n = substr(s, 1, RLENGTH-1) + 0; u = substr(s, RLENGTH, 1)
    t += n * (u=="D" ? 86400 : u=="H" ? 3600 : u=="M" ? 60 : 1)
    s = substr(s, RLENGTH+1)
  }
  return t
}
BEGIN {
  print "date,job,pid,workers_running,workers_failed,items_submitted,items_inflight,results_received,sub_results_received,errors,time_since_submitted,time_since_first_worker,time_since_last_result,throughput"
}
{
  d = $1; task = $2; pid = $3; rest = $4
  gsub(/[wirt]:/, "", rest)
  split(rest, a, / - /)
  split(a[1], w, "/"); split(a[2], i, "/")
  split(a[3], r, "/"); split(a[4], t, "/")

  x = d; gsub(/[-:]/, " ", x); ts = mktime(x)
  thr = 0
  if ((pid in pfl) && ts > pts[pid]) thr = (pfl[pid] - i[2]) / (ts - pts[pid])
  pfl[pid] = i[2]; pts[pid] = ts

  printf "%s,%s,%s,%d,%d,%d,%d,%d,%d,%d,%d,%d,%d,%.2f\n",
    d, task, pid, w[1], w[2], i[1], i[2], r[1], r[2], r[3],
    secs(t[1]), secs(t[2]), secs(t[3]), thr
}'
