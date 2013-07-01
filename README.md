libtraceproc - a library for tracing [Pro*C][proc] and [OCI][oci] calls

It is implemented as shared library which intercepts Pro*C/OCI
calls when preloaded. Since DB libraries like [OCCI][occi] and
[OTL][otl] also use low-level OCI-calls, one get useful trace
results there, too.

In Pro*C trace mode it reconstructs SQL statements using bound
host-variables.

libtraceproc may help to answer questions like these:

- What SQL-statements are executed by an existing code-base (i.e.
  including all used libraries)?
- What are the actual host-variable values of a failed SQL
  statement?
- To which OCI calls map certain Pro*C constructs?
- What OCI calls are made by a given OTL 'generated' code?
- Has the MAX_ROW_INSERT option (implicitly buffered inserts) the
  intended effect?
  (only if it is set for the translation unit that contains the embedded
  SQL connect statement)
- Which select-cursors are using what fetch sizes (e.g. via PREFETCH
  option or via explicit array bulk fetches)?
- Are array-based bulk-inserts/selects used?
- Does my OCI program contains some handle/descriptor leaks?

Using libtraceproc for tracing SQL statements (including host
variable values) before and/or after the actual execution frees
the developer from the tedious exercise of adding  boilerplate
code to a DB client program for providing a comparable - say -
verbose mode.


## Examples


### Pro*C

    $ make example/massinsert
    
    $ TRACEPROC_OPTIONS="-help" LD_PRELOAD=./libtraceproc.so ./example/massinsert
    libtraceproc - a tracing library for Oracle Pro*C
    
    Call: $ TRACEPROC_OPTIONS="-opt1 ..." LD_PRELOAD=/path/libtraceproc.so
    
    Options:
    
      -help        - this screen
      -fFILENAME   - write trace to FILENAME (default: stderr)
      -tFORMAT     - timestamp strftime-like string (default: %F_%H:%M:%S.#S:#_)
      -intercept   - enable Pro*C tracing (default: no)
      -oci         - also trace OCI calls (default: no)
    [..]
    
    $ TRACEPROC_OPTIONS="-intercept -notime" LD_PRELOAD=./libtraceproc.so < msmall.inp ./example/massinsert
    Libtraceproc is active.
    TRACEPROC_OPTIONS='-intercept -notime'
    
    
    -- ###########################################################################
    -- Before execution:
    -- example/massinsert.pc:158
    -- MAX_ROW_INSERT=0 (max # of implicitly buffered rows on INSERTs)
    CONNECT 'juser' IDENTIFIED BY 'geheim' USING 'orcl';
    -- ---------------------------------------------------------------------------
    
    [..]
    
    -- Before execution:
    -- example/massinsert.pc:109
    insert into tageswerte_tbl (station_id,meassure_date,quality,bedeckungsgrad,rel_feuchte,dampfdruck,lufttemperatur,luftdruck_stationshoehe,windgeschwindigkeit,lufttemp_am_erdb_min,lufttemp_min,lufttemp_max,windspitze_max,niederschlagshoehe_ind,niederschlagshoehe,sonnenscheindauer,schneehoehe) values ('2290' ,'17810103' ,'2' ,'-999' ,'-999' ,'-999' ,'-4.7' ,'890.40' ,'-999' ,'-999' ,'-999' ,'-999' ,'-999' ,'0' ,'.0' ,'-999' ,'-999' );
    
    [..]
               STATEMENT |      COUNT |       TIME (s) |    %
    ---------------------+------------+----------------+-----
      DROP TBL|SYN|ALTER |          2 |          0.033 | 0.20
                  INSERT |         10 |          0.015 | 0.09
                 CONNECT |          1 |          0.035 | 0.21
                  COMMIT |          1 |          0.003 | 0.02
          COMMIT RELEASE |          1 |          0.005 | 0.03
    CREATE TABLE|SYNONYM |          1 |          0.075 | 0.45
    =====================+============+================+=====
                     SUM |         16 |          0.166 | 1.00
    
    
    
                FUNCTION |      COUNT |       TIME (s) |    %
    ---------------------+------------+----------------+-----
                  sqlcxt |         16 |          0.166 | 0.98
                 sqlorat |         16 |          0.000 | 0.00
                 non-sql |          x |          0.003 | 0.02
    =====================+============+================+=====
                     SUM |         32 |          0.169 | 1.00


### OTL

    $ make example/otl/fetch
    
    $ TRACEPROC_OPTIONS="-oci -nosql -gory -notime " LD_PRELOAD=./libtraceproc.so .inp ./example/otl/fetch
    OCIInitialize(OCI_DEFAULT) => OCI_SUCCESS (0)
    OCIEnvInit(OCI_DEFAULT) => OCI_SUCCESS (0)
    OCIHandleAlloc(HANDLE_000 type=2 0x18c6818) => OCI_SUCCESS (0)
    OCIHandleAlloc(HANDLE_001 type=8 0x18c7700) => OCI_SUCCESS (0)
    OCIHandleAlloc(HANDLE_002 type=3 0x18c6738) => OCI_SUCCESS (0)
    OCIServerAttach(orcl OCI_DEFAULT) => OCI_SUCCESS (0)
    OCIAttrSet(OCI_ATTR_SERVER) => OCI_SUCCESS (0)
    OCIHandleAlloc(HANDLE_003 type=9 0x19025e0) => OCI_SUCCESS (0)
    OCIAttrSet(OCI_ATTR_USERNAME juser) => OCI_SUCCESS (0)
    OCIAttrSet(OCI_ATTR_PASSWORD geheim) => OCI_SUCCESS (0)
    OCISessionBegin(OCI_CRED_RDBMS, OCI_DEFAULT) => OCI_SUCCESS (0)
    [..]
    OCIStmtExecute(iters=0, rowoff=0, OCI_DEFAULT) => OCI_SUCCESS (0)
    OCIStmtFetch(rows=1024, OCI_FETCH_NEXT) => OCI_NO_DATA (100)
    OCIAttrGet(OCI_ATTR_ROW_COUNT) => OCI_SUCCESS (0)
    [..]
                FUNCTION |      COUNT |       TIME (s) |    %
    ---------------------+------------+----------------+-----
              OCIEnvInit |          1 |          0.001 | 0.01
          OCIStmtPrepare |          1 |          0.000 | 0.00
          OCIDefineByPos |          2 |          0.000 | 0.00
          OCIStmtExecute |          2 |          0.001 | 0.03
            OCIStmtFetch |          1 |          0.001 | 0.02
          OCIHandleAlloc |          6 |          0.000 | 0.00
           OCIHandleFree |          7 |          0.000 | 0.00
         OCISessionBegin |          1 |          0.019 | 0.44
           OCISessionEnd |          1 |          0.003 | 0.06
              OCIAttrGet |         14 |          0.000 | 0.00
              OCIAttrSet |          4 |          0.000 | 0.00
         OCIServerAttach |          1 |          0.014 | 0.33
         OCIServerDetach |          1 |          0.000 | 0.00
           OCIInitialize |          1 |          0.003 | 0.07
             OCIParamGet |          2 |          0.000 | 0.00
                 non-sql |          x |          0.001 | 0.02
    =====================+============+================+=====
                     SUM |         45 |          0.042 | 1.00


## Example directory 

The sub-directory `example` contains some small example programs
for testing the tracing features of libtraceproc, all compilable
via the makefile.

But be careful, it also contains 3 example programs that trigger
Pro*C bugs - under `example/bug`.

## Compile

    $ make

Some unit-tests:

    $ make testunit

(the 2nd suite needs an configured Oracle DB instance)

Some module-tests (need Oracle DB):

    $ make testmod

The DB related tests expect the environment variables
ORACLE_USER, ORACLE_PASS, ORACLE_SID to be set and a writable
user schema for temporary tables.

## Setup

To trace anything you have to specify the shared-library
`libtraceproc.so` in the `LD_PRELOAD` environment variable. By
default nothing is traced, you have to enable tracing via adding
`-intercept` or `-oci` to the TRACEPROC_OPTIONS environment
variable. `-help` prints all available options.


## Environment

A somehow recent C Compiler that supports some C99 features and a
few [GCC][gcc]-style attributes should be fine. The makefile uses
[GNU make][gmake] features. GNU sed is needed for the module
tests.

Tested with:

- CentOS 6.4, Oracle 11g Standard Edition 11.2.0.1, GCC 4.4.7,
  GNU make 3.81

Should also run on Solaris (with Solaris Studio >= 12.3 or an GCC
compiler and GNU make).


## Advanced features

- one can set a (conditional) breakpoint on `traceproc_trap()` which
  demultiplexes all traced calls
- libtraceproc_dummy.so can be normally linked to a project for
  registering custom callback functions (e.g. for custom pretty printing).
  Then only when a preloaded real libtraceproc.so is available those
  callbacks are enabled.
- ret_check.h, ora_util.h, oci_util.h, proc_util.h contain some
  functions/macros that could be useful for general purpose DB
  related programs

## License

[GPLv3+][gpl]

## Contact

Don't hesitate to send me comments and feedback via email:

    Georg Sauthoff <mail@georg.so>


[proc]:  http://en.wikipedia.org/wiki/Pro*C
[oci]:   http://en.wikipedia.org/wiki/Oracle_Call_Interface
[occi]:  http://en.wikipedia.org/wiki/Oracle_C++_Call_Interface
[otl]:   http://otl.sourceforge.net/
[gmake]: http://www.gnu.org/software/make/
[gcc]:   http://gcc.gnu.org/
[gpl]:   http://www.gnu.org/licenses/gpl.html
