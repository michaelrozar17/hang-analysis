# hang-analysis
Script(s) on hang-analysis.
The data dumped in hang-analysis trace file  (generated by Oracle during database hung) are in key-value pairs. This Perl script reads 
the trace file and output top N data in a easy-to-read tabular format under following sections:

Sort by Max no. of Blocking Sessions,Sort by Max Time in Wait and Sort by Max Running time and Not in Wait (spinning on CPU or hung)

Running the script:

Download the script to any linux or your local windows machine and run by passing mandatory argument: --tracefile=/path/to/hang-analysis.trc.
Output is displayed on screen by default.

./hanalyzer.pl --help
./hanalyzer.pl --tracefile=/path/to/xzy2_ora_25059.trc  # reads trace file xzy2_ora_25059.trc, output is sent to console/screen

./hanalyzer.pl --tracefile=/path/to/xzy2_ora_25059.trc > /tmp/haoutput.txt # reads trace file xzy2_ora_25059.trc, output redirected to /tmp/haoutput.txt

./hanalyzer.pl --tracefile=/path/to/xzy2_ora_25059.trc --csv > /tmp/haoutput.txt # generates output in a csv format 
./hanalyzer.pl --tracefile=/path/to/xzy2_ora_25059.trc --display= 30 # limits output to 30 rows per section, default is 15. Note not always top n results are the cause of a problem, sometimes sessions showing-up in bottom can be the culprit


Output Explained:

The first section: Sort by Max no. of Blocking Sessions - lists the session that blocks most number of sessions. In the sample below output, session with OS PID 223332 in the top blocks 17 other sessions. Further down the output, OS PID 167211 and 107636 is blocked by this max blocking session 223332.

Sort by Max no. of Blocking Sessions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Inst|    OS_ID|  ProcessID|  SessionID|  SerialID| WaitingOn                     | TimeInWait(sec)| Blocking_Sess_Cnt| Current_SQL                   | BlockedBy
------------------------------------------------------------------------------------------------------------------------------------------------------------------
   4|   223332|        534|        414|     43746| PX Deq: Join ACK              |        1.389201|                17| SELECT COUNT(1) FROM PA_USUARI| NULL
   4|   223332|        534|        414|     43746| PX Deq: Join ACK              |        0.226484|                17| SELECT COUNT(1) FROM PA_USUARI| NULL
   4|   335914|         98|        148|     55352| cursor: pin S wait on X       |       15.012479|                 0| SELECT COUNT(1) FROM PA_USUARI| NULL
   4|   167211|        339|        202|     19957| cursor: pin S wait on X       |       15.008163|                 0| SELECT COUNT(1) FROM PA_USUARI| instance: 4, os id: 223332, session id : 414
   4|   107636|        389|        330|      8609| cursor: pin S wait on X       |       15.022098|                 0| SELECT COUNT(1) FROM PA_USUARI| instance: 4, os id: 223332, session id : 414

The output is continued with the call stack information in the same order as above i.e call stack of OS PID 223332,223332,335914. The output within bracket () in the last column below shows the wait events coming under "wait history:"

Continuing with Stack information..

   4|   223332|        534|        414| ksedsts ksdxfstk ksdxcb sspuser sighandler poll sskgxp_selectex skgxpiwait skgxpwaiti skgxpwait ksxpwait ksliwat kslwaitctx ksxprcv_int ksxprcvimdwct
x kxfpqidqr kxfprienq kxfpSendJoin kxfpg1sg kxfpgsg kxfrAllocSlaves kxfrialo kxfralo qerpx_rowsrc_start qerpxStart rwsstd qergsStart rwsstd selexe0 opiexe | (PX Deq: reap credit PX Deq: Join ACK PX Deq: reap credit)
   4|   223332|        534|        414| ksedsts ksdxfstk ksdxcb sspuser sighandler poll sskgxp_selectex skgxpiwait skgxpwaiti skgxpwait ksxpwait ksliwat kslwaitctx ksxprcv_int ksxprcvimdwct
x kxfpqidqr kxfprienq kxfpSendJoin kxfpg1sg kxfpgsg kxfrAllocSlaves kxfrialo kxfralo qerpx_rowsrc_start qerpxStart rwsstd qergsStart rwsstd selexe0 opiexe | (PX Deq: reap credit PX Deq: Join ACK PX Deq: reap credit)
   4|   335914|         98|        148| ksedsts ksdxfstk ksdxcb sspuser sighandler select skgpwwait kgxWait kgxSharedExamine kxsGetRuntimeLock kkscsCheckCursor kkscsSearchChildList kksfbc kkspbd0 kksParseCursor opiosq0 opipls opiodr rpidrus skgmstack rpiswu2 rpidrv psddr0 psdnal pevm_EXECC pfrinstr_EXECC pfrrun_no_tool pfrrun plsql_run peicnt | (cursor: pin S wait on X cursor: pin S wait on X cursor: pin S wait on X)

The "Sort by Max Time in Wait" section as the name implies orders the output by maximum time in wait. In the following output session with OS PID 78215 has waited for 1944 seconds followed by os pid 78217 for 1943 seconds. Call stack information for corresponding sessions is also displayed.

Sort by Max Time in Wait
~~~~~~~~~~~~~~~~~~~~~~~~
Inst|    OS_ID|  ProcessID|  SessionID|  SerialID| WaitingOn                     | TimeInWait(sec)| Blocking_Sess_Cnt| Current_SQL                   | BlockedBy
------------------------------------------------------------------------------------------------------------------------------------------------------------------
   4|    78215|        437|        352|     47369| Redo Transport Open           |            1944|                 0| <none>         | NULL
   4|    78217|        494|        924|     25943| Redo Transport Open           |            1943|                 0| <none>         | NULL


3rd section "Sort by Max Running time and Not in Wait (spinning on CPU or hung)" - display those sessions that are running on CPU or hung state (and not waiting). In one of the SR, Ct was complaining about hung sessions -  the trace file did not show notable "blocking" or "time in wait" sessions but still sesions were hung. This is when we came to know sessions can also go spinning on cpu. "last wait:" column in trace file helps to get this information.
Session with os pid 16596 was running for 288 seconds without waiting i.e ON CPU as seen below.

Sort by Max Running time and Not in Wait (spinning on CPU or hung)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Inst|    OS_ID|  ProcessID|  SessionID|  SerialID| WaitingOn                     | RunningFor(sec)| Blocking_Sess_Cnt| Current_SQL
-------------------------------------------------------------------------------------------------------------------------------------------------
   1|    16596|        272|       2834|     39305| Not in Wait                   |             288|                 0| SELECT /*+all_rows*/ SYS_XMLGE
   1|    16596|        272|       2834|     39305| Not in Wait                   |       47.546243|                 0| SELECT /*+all_rows*/ SYS_XMLGE

Last we summarize and display all the wait history

Top WaitEvents History Count
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
control file sequential read       |    18
REPL Capture/Apply: messages       |     6
latch: cache buffers chains        |     3
flashback log file sync            |     1
db file sequential read            |     1
direct path write                  |     1
     
