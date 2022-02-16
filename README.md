# Row Level Security in the Oracle Database using IAM User Authentication
Sample scripts for configuration of row level security in the database using IAM users and groups

```
<copy>
-------------------------------------------
-- Create and apply security policy (everything created as admin user)
-- This is not strictly needed, but does put values in memory rather than querying them over and over
-------------------------------------------

CREATE or REPLACE PACKAGE demo_security_context IS 
PROCEDURE  prc_set_security;
PROCEDURE  whoami (v_userid varchar2;
END;
/

CREATE or REPLACE PACKAGE BODY demo_security_context IS
--
PROCEDURE prc_set_security IS 
  v_predicate varchar2(1000) := '''UK'',''US''';
  BEGIN
  DBMS_SESSION.SET_CONTEXT('demo_security', 'country_id', v_predicate);
END prc_set_security;
--
PROCEDURE whoami (v_userid varchar2) IS 
  BEGIN
  DBMS_SESSION.SET_CONTEXT('demo_security', 'userid', v_userid);
END whoami;
--
END demo_security_context;
/

CREATE OR REPLACE CONTEXT demo_security USING demo_security_context;

grant execute on demo_security_context to public;

------------------------
-- Set Security Context - you typically do this in a login trigger
------------------------

exec admin.demo_security_context.prc_set_security;

-- Test to ensure security context is set
select sys_context('demo_security','country_id') from dual;

'US','UK'

------------------------
-- Code snippit to build predicate looping through all the values
------------------------

set serveroutput on

declare
v_predicate varchar2(4000) :='(';
begin
for i in (select distinct country_id from locations)
loop
v_predicate := v_predicate||''''||i.country_id||''',';
end loop;
v_predicate := rtrim(v_predicate,',')||')';
dbms_output.put_line(v_predicate);
end;
/

('CN','CH','NL','JP','SG','DE','MX','IT','US','UK','IN','AU','BR','CA')

------------------------
-- create predicate function
------------------------

-- for everyone who is NOT derrick.cameron limit rows to US and CN per the system context (above) else return all rows.

CREATE OR REPLACE FUNCTION apply_demo_policy_fn (obj_schema VARCHAR2, obj_name VARCHAR2) RETURN VARCHAR2 IS
  d_predicate VARCHAR2(2000);
  BEGIN
    IF sys_context('demo_security','userid') <> 'oracleidentitycloudservice/adw_demo_user' THEN
    d_predicate := 'country_id IN ('||sys_context('demo_security','country_id')||')';
    ELSE
    d_predicate := '1=1';
  -- d_predicate := 'country_id in (select country_id from demo.locations_user_access)';
   END IF;
RETURN D_predicate;
END apply_demo_policy_fn;

------------------------
-- add policy to table
------------------------

BEGIN
DBMS_RLS.ADD_POLICY (
	object_schema	=> 'demo', 
	object_name	=> 'locations', 
	policy_name	=> 'demo_security_policy', 
	function_schema	=> 'admin',
	policy_function	=> 'apply_demo_policy_fn', 
	statement_types => 'select');
END;

EXECUTE DBMS_RLS.DROP_POLICY ('demo', 'locations', 'demo_security_policy')

------------------------
-- create login trigger
------------------------

CREATE OR REPLACE TRIGGER security_init_trg 
AFTER LOGON
ON DATABASE
BEGIN
   admin.demo_security_context.prc_set_security;
   admin.demo_security_context.whoami(sys_context('userenv','authenticated_identity'));
END;
/

select * from demo.locations (DO NOT JUST CLICK ON THE TABLE IN SQLDEVELOPER - YOU WILL GET ORASCN ERROR)

select apply_demo_security.apply_demo_policy_fn('demo','locations') from dual;

------------------------
-- procedure to call in the rpd
------------------------

CREATE OR REPLACE PROCEDURE security_init_prc (p_userid in varchar2) IS
BEGIN
--if p_userid <> 'adw_demo_user' then
admin.demo_security_context.prc_set_security;
admin.demo_security_context.whoami(p_userid);
commit;
--end if;
end;
/

exec admin.security_init_prc('derrick.cameron@oracle.com');

grant execute on security_init_prc to public;
</copy>
```