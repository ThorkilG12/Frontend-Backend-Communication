## Who did the update of a table ?

Setup: A windows server running apache, php and postgresql

When php connects to the database it uses a user called "apache". This user exists in postgresql, and the access to different tables and objects is given to user "apache".
At the same time I have users logging into my server, and I am setting the userId and the mail-address for the user in the PHP session.

So someone@somewhere.org is logged into my web-server, but PostgreSQL only see that user "apache" is inserting, reading and updating.

### Q: How to pass the mail-address to the postgreSQL server ?

The solution:
in PHP I have this: `include_path = "d:\apache\common"`
All PHP modules that connects to PostgreSQL is using `require_once 'pdo.php';`

In pdo.php I have this 
```PHP
if (isset($_SESSION['userMail'])) {
  $sql = "CREATE TEMPORARY TABLE current_app_user (username text);
	          INSERT INTO current_app_user (username) VALUES ('" . $_SESSION['userMail'] . "');";
	if (!($pdo->exec($sql))) {echo "e:NewPDO Failed";} 
}
``` 
If a users is logged in (having a value in `$_SESSION['userMail']`), then a temporary table is created in Postgres, and this table holds one row with one column (username).

In the database I have a function called `get_app_user()`
```SQL
CREATE OR REPLACE FUNCTION public.get_app_user()
 RETURNS text
 LANGUAGE plpgsql
AS $function$
DECLARE
    cur_user text;
BEGIN
    BEGIN
        cur_user := (SELECT username FROM current_app_user);
    EXCEPTION WHEN undefined_table THEN
        cur_user := 'unknown_user';
    END;
    RETURN cur_user;
END;
$function$
;
```
This function is used by all tables in the database that want's to know the mail of the loggedIn user.

Let me use my log table as an example. It has a trigger that calls a function:
```SQL
create trigger t_log_trigger_1 after
insert
    or
update
    of event on
    public.t_log for each row execute function t_log_func_1()
;

CREATE OR REPLACE FUNCTION public.t_log_func_1()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$ 
     BEGIN
      UPDATE t_log
      SET modified_by = get_app_user(),
        modified_by_session_user = SESSION_USER,
        modified = now(),
	      backend_pid = pg_backend_pid()
      WHERE id = NEW.id ;
      RETURN NEW;END; 
      $function$
;    
```
