# utl-partitioning-your-table-for-a-big-sort
Partitioning your table for a big sort.
    Partitioning your table for a big sort

    see SAS Forum
    https://tinyurl.com/y9rrqngr
    https://communities.sas.com/t5/SAS-Programming/Proc-Sql-and-Proc-Sort-execution-failure/m-p/501385

    Partitioning your table using the mod function for a big sort.

    The method you choose depends almost entirely on the locality of keys, cardinality of keys
    skewness of keys and width of record.

    You can partition in many ways ie State, Age, Years. They do not have to be part of the sort
    keys.

    If the data is highly clustered on keys an index on the big table may be better.

    Make your own grid computing application on a power workstation(SAS calls it a PC)

    The method you choose depends almost entirely on the locality of keys, cardinality of keys
    and skewness of keys.

    One Technique:
    ==============

      If you have a high cardinality uniform integer variable on a moderate table(~100gb to 1tb) then you can
      partition the data and run parallel or serial tasks.
      You can also use firstobs and obs

      For smaller data ie <100gb a the multi-threaded sort task may be optimum.

      For 8 mutually exclusive partitions use

      Note these are mutually exclusive Partitons (can also use first obs and obs)

          data(where=(mod(large_integer,8)=1))
          data(where=(mod(large_integer,8)=2))
         ...
          data(where=(mod(large_integer,8)=0))


      If you sort each partition anf the numeric value is the key
       then you don't need a by statement to put the pieces back together
      just use a view. This may be faster than a single physical dataset?

      Even if you have a by statement to interleave use a view?


    HAVE 80,000,000 random uniforms
    ===============================

    Up to 40 obs SD1.BIGDATA total obs=80,000,000

     KEY      RAN

       1    0.62847
       2    0.63653
       3    0.30777
       4    0.29759
       5    0.80438
       6    0.74035
       7    0.58628
       8    0.34328
       9    0.87528
    ...

    WANT 80,000,000 sorted

    Up to 40 obs SD1.WANT total obs=80,000,000

        KEY           RAN

      9384312     0.0000012847
     76198904     0.0000023653
     76140192     0.0000030777
     59744648     0.0000049759
      3617920     0.0000050438
      5048352     0.0000064035
     26856808     0.0000079728
     75829272     0.0000089728
     66415312     0.0000099728


    WORKING CODE (run simultaneously or sequentially - restricts the bigest temp table)

       proc sort data = sd1.bigdata(where=(mod(key,8)=1)) out= sd1.a1 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=2)) out= sd1.a2 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=3)) out= sd1.a3 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=4)) out= sd1.a4 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=5)) out= sd1.a5 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=6)) out= sd1.a6 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=7)) out= sd1.a7 noequals;
       proc sort data = sd1.bigdata(where=(mod(key,8)=0)) out= sd1.a0 noequals;


    FULL SOLUTION

    * create some data;
    libname sd1 "d:/sd1";
    data sd1.bigdata;
     do key=1 to 800;
       ran=uniform(-1);
       output;
     end;
    run;

    * SAVE the program in c:/oto;
    data _null_;file "c:\oto\getmode.sas" lrecl=512;input;put _infile_;putlog _infile_;
    cards4;
    %macro getmode(remainder);
      libname sd1 "d:/sd1";
      proc sort data = sd1.bigdata(where=(mod(key,8)=&remainder)) out= sd1.a&remainder noequals;
      by ran;
      run;quit;
    %mend getmode;
    ;;;;
    run;quit;

    * test the macro interactively;
    * note you can highlight and hit RMB(submit) to compile macro in your interactive session
      then highlight and RMB the code below to test;

    %getmode(0);

    %let _s=%sysfunc(compbl(C:\Progra~1\SASHome\SASFoundation\9.4\sas.exe -sysin
    c:\nul -sasautos c:\oto -autoexec c:\oto\Tut_Oto.sas
    -work d:\wrk));

    * The argument of getmode is the remainder after dividing by 8;

    options noxwait noxsync;
    %let tym=%sysfunc(time());
    systask kill sys1 sys2 sys3 sys4  sys5 sys6 sys7 sys8;
    systask command "&_s -termstmt %nrstr(%getmode(1);) -log d:\log\a1.log" taskname=sys1;
    systask command "&_s -termstmt %nrstr(%getmode(2);) -log d:\log\a2.log" taskname=sys2;
    systask command "&_s -termstmt %nrstr(%getmode(3);) -log d:\log\a3.log" taskname=sys3;
    systask command "&_s -termstmt %nrstr(%getmode(4);) -log d:\log\a4.log" taskname=sys4;
    systask command "&_s -termstmt %nrstr(%getmode(5);) -log d:\log\a5.log" taskname=sys5;
    systask command "&_s -termstmt %nrstr(%getmode(6);) -log d:\log\a6.log" taskname=sys6;
    systask command "&_s -termstmt %nrstr(%getmode(7);) -log d:\log\a7.log" taskname=sys7;
    systask command "&_s -termstmt %nrstr(%getmode(0);) -log d:\log\a8.log" taskname=sys8;
    waitfor sys1 sys2 sys3 sys4  sys5 sys6 sys7 sys8;
    %put %sysevalf( %sysfunc(time()) - &tym);?

    * VIEW to put back together;
    data humptyback/view=humptyback;
      set sd1.a1 sd1.a2 sd1.a3 sd1.a4 sd1.a5 sd1.a6 sd1.a7 sd1.a0;
      /* you do need the by statement for the mod technique */
      /* no by statement needed if pieces sorted  ie piece1 0<=key<=1000000, 1000001<=piece2<=2000000*/
      /* cand be faster than one physical datasets depending on where the pieces are stored */
    run;quit;

    * from a1.lod
    NOTE: There were 100 observations read from the data set SD1.BIGDATA.
          WHERE MOD(key, 8)=1;
    NOTE: The data set SD1.A1 has 100 observations and 2 variables.
    NOTE: PROCEDURE SORT used (Total process time):
          real time           0.01 seconds
          cpu time            0.00 seconds


        KEY           RAN

      9384312     0.0000012847
     76198904     0.0000023653
     76140192     0.0000030777
     59744648     0.0000049759
      3617920     0.0000050438
      5048352     0.0000064035
     26856808     0.0000079728
     75829272     0.0000089728
     66415312     0.0000099728


