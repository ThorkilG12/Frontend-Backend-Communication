## Who did the update of a table ?

Setup: A `windows 2019 server` running `apache 2.4`, `php 8` and `postgresql 18`

I my setup I know the email of my authenticated users. This information is in a session variable named "usermail".

Everytime PHP want to connect to PostgreSQL, it is done by this PHP statement: `require_once 'pdo.php';`

in the setup of PDO, I also have this:
```PHP
if (isset($_SESSION['userMail'])) {
	$sql = "CREATE TEMPORARY TABLE current_app_user (username text);
	INSERT INTO current_app_user (username) VALUES ('" . $_SESSION['userMail'] . "');";
	$pdo->exec($sql);
}	
```
In the PostgreSQL template `template1` I have this function:
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
By having it in the tamplate1 database, all new databases will have it as well.

To handle activity on my server, I choose between two options:

### 1) Tracking who and when a row is created or updated
Four new columns are added to the table:
```SQL
	created timestamptz DEFAULT now() NULL,
	modified_by varchar(63) NULL,
	modified_by_session_user varchar(63) NULL,
	modified timestamptz NULL
```
A trigger is created:
```SQL
create trigger <tablename>_trigger_1 after
insert or update of
	var1, var2..... varx
on
    public.<tablename> for each row execute function <tablename>_func_1()
```
The function looks like this:
```SQL
CREATE OR REPLACE FUNCTION public.<tablename>_func_1()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$ 
     BEGIN
      UPDATE public.<tablename>
      SET modified_by = get_app_user(),
      modified_by_session_user = SESSION_USER,
      modified=now()
      WHERE <row id> = NEW.<row id> ;
      RETURN NEW;END; 
      $function$
;
```
### 2) Saving information about changes in a history table
Like before, I put this function into `template1`:
```SQL
CREATE OR REPLACE FUNCTION public.track_changes()
RETURNS trigger
LANGUAGE plpgsql
AS $function$
DECLARE
    v_history_table TEXT;
    v_action_type TEXT;
    v_user TEXT := session_user;
BEGIN
    v_history_table := TG_TABLE_NAME || '_history';
    v_action_type := TG_OP;
    IF (v_action_type = 'UPDATE' OR v_action_type = 'DELETE') THEN
        EXECUTE format('
            INSERT INTO public.%I
            SELECT 
                ($1).*,
                nextval(''%I_history_id_seq''),
                %L AS action_type,
                %L AS modified_by_session_user,
                NOW() AS modified,
                get_app_user() as modified_by;',
            v_history_table,               -- %I (history table name)
            TG_TABLE_NAME,                 -- %I (sequence name construction)
            v_action_type,                 -- %L (action_type)
            v_user                         -- %L (session_user)
        )
        USING OLD;
    ELSIF (v_action_type = 'INSERT') THEN
        EXECUTE format('
            INSERT INTO public.%I
            SELECT 
                ($1).*,
                nextval(''%I_history_id_seq''),
                %L AS action_type,
                %L AS modified_by_session_user,
                NOW() AS modified,
                get_app_user() as modified_by;',
            v_history_table,               -- %I (history table name)
            TG_TABLE_NAME,                 -- %I (sequence name construction)
            v_action_type,                 -- %L (action_type)
            v_user                         -- %L (session_user)
        )
        USING NEW;
    END IF;
    IF (v_action_type = 'DELETE') THEN
        RETURN OLD; -- Returning OLD allows the DELETE to proceed.
    ELSE
        RETURN NEW; -- Returning NEW allows the INSERT/UPDATE to proceed.
    END IF;
END;
$function$;
```

In the main table, I have this column: `id int generated always as identity`. It will make PostgreSQL to create a sequence like:
Create the history table like this: `<tablename>_history_id_seq`
```SQL
CREATE TABLE public.<tablename>_history AS
TABLE public.<tablename> WITH NO DATA;
-- Add history-specific tracking columns
ALTER TABLE public.<tablename>_history
ADD COLUMN history_id BIGSERIAL PRIMARY KEY, -- Unique ID for the history record
ADD COLUMN action_type VARCHAR(10) NOT NULL, -- 'INSERT', 'UPDATE', or 'DELETE'
ADD COLUMN modified_by_session_user TEXT,                 -- User who made the change
ADD COLUMN modified TIMESTAMP WITH TIME ZONE DEFAULT NOW(), -- Timestamp of the change
ADD COLUMN modified_by TEXT; --
```
In the table for that active data create this trigger:
```SQL
create or replace trigger <tablename>_history_trigger 
before insert or delete or update
on public.<tablename> 
for each row execute function track_changes()
```
Thorkil - Oct 2025
