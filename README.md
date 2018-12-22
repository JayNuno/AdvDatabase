# AdvDatabase
College Enrollment System using PL/SQL

--Creating a deployable package
create or replace package Enroll is

function check_snum(p_snum varchar2) return varchar2;

function check_callnum(p_callnum varchar2) return varchar2;

function double_enrollment(p_snum varchar2, p_callnum varchar2) return varchar2;

function check_section(p_snum varchar2, p_callnum varchar2) return varchar2;

function check_15credit_rule(p_snum varchar2, p_callnum varchar2) return varchar2;

function check_student_standing(p_snum varchar2, p_callnum varchar2) return varchar2;

function disqualified_student(p_snum varchar2) return varchar2;

function capacity_limit(p_callnum varchar2) return varchar2;

function repeat_waitlist(p_snum varchar2, p_callnum varchar2) return varchar2;

procedure addme_proj(p_snum number, p_callnum number, p_out varchar2);

function not_enrolled_dropme(p_snum varchar2, p_callnum varchar2) return varchar2;

function already_graded(p_snum varchar2, p_callnum varchar2) return varchar2;

function drop_enrollments(p_snum varchar2, p_callnum varchar2) return varchar2;

procedure dropme_proj(p_snum number, p_callnum number);

end Enroll;
/

create or replace package body Enroll is 

--Checks to see if snum exist(COMPLETE)
function check_snum(p_snum varchar2) 

   return varchar2 is

   v_snum number(3);


begin 

select count(snum) into v_snum
 from students 
 where snum = p_snum;



   IF v_snum = 1 then  
       return null;
   else 
       return ('Student number ' || p_snum || ' is not a student (function)');
   end if;

end;


--Checks to see if callnum exist(COMPLETE)
function check_callnum(p_callnum varchar2)
   return varchar2 is 

   v_callnum number(10);

begin 
   
   select count(callnum) into v_callnum
   from schclasses 
   where callnum = p_callnum;

   IF v_callnum = 1 THEN 
       return null;
   else 
       return ('Class number ' || p_callnum || ' is not a class (function)');
   end if;

end;


function double_enrollment(p_snum varchar2, p_callnum varchar2)
   return varchar2 is 

   v_callnum_enrolled varchar2(200);

BEGIN 
   
   --CHECKS TO SEE IF STUDENT IS ENROLLED IN THE CLASS 
   select count(callnum) into v_callnum_enrolled
   from enrollments 
   where callnum = p_callnum
   and snum = p_snum;

   --IF THEY ARENT ALREADY ENROLLED
   IF v_callnum_enrolled = 0 THEN 
       return null;
   ELSE 
       return ('Student number ' || p_snum || ' is already enrolled in ' || p_callnum || ' (ran in function)');
   END IF; 

END;



--CHECKS FOR REPEAT ENROLLMENT IN SECTION 
function check_section(p_snum varchar2, p_callnum varchar2)
return varchar2 is 

v_snum varchar2(200);

BEGIN
   

   --CHECKS TO SEE IF SAME SECTION OF CALLNUM
   select count(snum) into v_snum
   from enrollments e, schclasses sc, schclasses sc2
   where e.snum = p_snum
   and sc2.callnum = p_callnum
   and e.callnum = sc.callnum
   and sc.dept = sc2.dept 
   and sc.cnum = sc2.cnum 
   and e.callnum != sc2.callnum;

   IF v_snum > 0 THEN
       return ('ERROR: you may not add another class of the same callnum (FUNCTION)');
   ELSE 
       return null; 
   END IF;


END;




--CHECKS TO SEE IF STUDENT EXCEEDS 15 CREDIT UNITS (COMPLETE)
function check_15credit_rule(p_snum varchar2, p_callnum varchar2)
   return varchar2 is 

   v_class_credit number(20);
   v_student_credit number (20);

BEGIN

   --CHECKS THE TOTAL CREDIT HOURS OF P_SNUM
   select sum(crhr) into v_student_credit
   from schclasses sc, enrollments e, courses c
   where sc.callnum = e.callnum 
   and sc.cnum = c.cnum
   and sc.dept = c.dept
   and snum = p_snum;

   --CHECKS FOR P_CALLNUM CREDIT HOURS
   select crhr into v_class_credit
   from schclasses sc, courses c
   where c.cnum = sc.cnum
   and c.dept = sc.dept
   and sc.callnum = p_callnum;

   IF (v_student_credit + v_class_credit) >= 15 then
       return 'ERROR student number ' || p_snum || ' has too many credits (IN FUNCTION)';
   ELSE 
       return null;
   END if;

END;


-- CHECK STANDING REQUIREMENT (COMPLETE)
function check_student_standing(p_snum varchar2, p_callnum varchar2)
   return varchar2 is 

   v_student_standing number(2);
   v_class_standing number(2);


BEGIN 

--CHECKING FOR STUDENTS STANDING/ IS ABLE TO ADD COURSE 
   select standing into v_student_standing  
   from students 
   where snum = p_snum;

   select standing into v_class_standing
   from courses c, schclasses sc
   where callnum = p_callnum
   and c.dept = sc.dept 
   and c.cnum = sc.cnum;

IF v_student_standing < v_class_standing THEN 
   return ('your class standing of ' || v_student_standing || ' does not meet the class standing requirement of ' 
       || v_class_standing || '(in function)');
ELSE 
   return null;
END IF; 

END;


--CHECK FOR STUDENTS WITH STANDING MORE THAN 1 AND GPA MORE THAN 2.0
function disqualified_student(p_snum varchar2)
   return varchar2 is 

   v_standing varchar2(200);
   v_gpa varchar2(200);


BEGIN 


   select standing into v_standing 
   from STUDENTS 
   where snum = p_snum;

   select gpa into v_gpa 
   from students 
   where snum = p_snum;


   IF v_standing > 1 and v_gpa < 2 THEN 
       return ('student number ' || p_snum || ' is on disqulified status and may not add (FUNCTION)');    
   ELSE 
       RETURN null;
   END IF; 


END; 






--CHECKS TO SEE IF THE CLASS LIMIT MAY ADD ANOTHER STUDENT
function capacity_limit(p_callnum varchar2)
   return varchar2 is 

   v_capacity number(3);
   v_class_capacity number(3);



BEGIN 


   -- FINDS THE CALLNUM capacity AND SUBTRACTS 1 SPACE
   select capacity into v_capacity 
   from schclasses 
   where callnum = p_callnum;

   select count(snum) into v_class_capacity
   from enrollments
   where callnum = p_callnum;

   IF (v_capacity - v_class_capacity) > 0 THEN  
       return null;
   ELSE 
       return ('ERROR: there is not enough room in ' || p_callnum || ' for you to add (in function)');
   END IF;


END;


function repeat_waitlist(p_snum varchar2, p_callnum varchar2)
   return varchar2 is 

v_snum varchar2(3);
v_callnum varchar2(9);

BEGIN 


   select count(snum) into v_snum
   from waitlist 
   where snum = p_snum
   and callnum = p_callnum; 

   

   IF v_snum = 1 THEN
       return (p_snum || ' is already on the waitlist for ' || p_callnum);
   ELSE 
       return null; 
   END IF; 


END; 



----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
------------------------------------- Main Program -------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------

procedure addme_proj(p_snum number, p_callnum number, p_out varchar2) AS


check_callnum_return varchar2(200);
check_snum_return varchar(200);
double_enrollment_return varchar2(200);
check_15credit_rule_return varchar2(200);
check_student_standing_return varchar2(200);
check_section_return varchar2(200);
disqualified_student_return varchar2(200);
capacity_limit_return varchar2(200);
repeat_waitlist_return varchar2(200);


BEGIN 



   --CHECK SNUM, CALLNUM

   check_callnum_return := check_callnum(p_callnum);
   check_snum_return := check_snum(p_snum);
   IF check_callnum_return is null and check_snum_return is null THEN 
       
       --CHECK FOR DOUBLE ENROLLMENT
       double_enrollment_return := double_enrollment(p_snum, p_callnum);
       IF double_enrollment_return is null THEN 
           
           --CHECKS TO SEE IF SNUM TOTAL CREDIT HOURS EXCEED 15
           check_15credit_rule_return := check_15credit_rule(p_snum, p_callnum);
           IF check_15credit_rule_return is null THEN 
           
               --CHECKS TO SEE IF STUDENT STANDING MEETS CLASS REQUIRED STUDENT STANDING
               check_student_standing_return := check_student_standing(p_snum, p_callnum);
               IF check_student_standing_return is null THEN 
                   
                   --CHECKS TO SEE IF STUDENT TRIES TO ADD SAME CNUM DIFFERENT CALLNUM
                   check_section_return := check_section(p_snum, p_callnum);
                   IF check_section_return is null THEN
                   
                       --CHECKS TO SEE IF STUDENT IS ON DISQUALIFIED STATUS
                       disqualified_student_return := disqualified_student(p_snum);
                       IF disqualified_student_return is null THEN 
                       
                           --CHECKS TO SEE IF THE CLASS CAPACITY IS ABLE TO ADD ANOTHER STUDENT
                           capacity_limit_return := capacity_limit(p_callnum);
                           IF capacity_limit_return is null THEN 
                               insert into enrollments values (p_snum, p_callnum, null);
                               dbms_output.put_line('CONGRATS! You are now enrolled!');
                           ELSE 
                               dbms_output.put_line(capacity_limit_return);
                               repeat_waitlist_return := repeat_waitlist(p_snum, p_callnum);
                               IF repeat_waitlist_return is null THEN 
                                   dbms_output.put_line('CONGRATS! you are now enrolled in the waitlist for ' || p_callnum);
                                   insert into waitlist values (p_snum, p_callnum, sysdate);
                               ELSE 
                                   dbms_output.put_line('You are already on the waitlist');
                               END IF;
                           END IF; 

                       ELSE 
                           dbms_output.put_line(disqualified_student_return);
                       END IF;

                   ELSE 
                       dbms_output.put_line(check_section_return);
                   END IF;

               ELSE 
                   dbms_output.put_line(check_student_standing_return);
               END IF;

           ELSE 
               dbms_output.put_line(check_15credit_rule_return);
           END IF; 

       ELSE 
           dbms_output.put_line(double_enrollment_return);
       END IF;

   ELSE 
       dbms_output.put_line('Sorry! You may not add (procedure)');
       dbms_output.put_line(check_callnum_return);
       dbms_output.put_line(check_snum_return);
   END IF;

END;








function not_enrolled_dropme(p_snum varchar2, p_callnum varchar2) 
   return varchar2 is 

   v_snum number(3);
   v_callnum number(9);

BEGIN 

   select count(snum) into v_snum
   from ENROLLMENTs 
   where snum = p_snum
   and callnum = p_callnum; 

   

   

   IF v_snum = 0 THEN
       return ('ERROR: you may not drop because ' || p_snum || ' not a student enrolled in this class (function)');
   ELSE 
       return null;
   END IF;

END;




function already_graded(p_snum varchar2, p_callnum varchar2)
   return varchar2 is 

   v_grade varchar2(3);


BEGIN 

   select grade into v_grade
   from enrollments 
   where snum = p_snum 
   and callnum = p_callnum;

   IF v_grade is null THEN 
       return null;
   ELSE 
       return ('ERROR: you may not drop because your records have a grade of ' || v_grade || ' for callnum of '
           || p_callnum) || '(FUNCTION)';
   END IF; 

END;


function drop_enrollments(p_snum varchar2, p_callnum varchar2)
   return varchar2 is 



BEGIN 


   update enrollments set grade = 'w' where snum = p_snum and callnum = p_callnum;
   if sql%found THEN 
       dbms_output.put_line('drop was successful');
       return null;
   else 
       return ('drop was unsuccessful');
   END IF; 


END; 







-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-----------------------------DROP ME MAIN PROGRAM------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------



procedure dropme_proj(p_snum number, p_callnum number) AS


   check_callnum_return varchar2(200);
   check_snum_return varchar2(200);
   not_enrolled_dropme_return varchar2(200);
   already_graded_return varchar2(200);
   drop_enrollments_return varchar2(200);
   waitlist_cursor_return varchar2 (200);
   p_out varchar2 (200);

BEGIN


   --CHECKS TO SEE IF SNUM, CALLNUM EXIST
   check_callnum_return := check_callnum(p_callnum); 
   check_snum_return := check_snum(p_snum);
   IF check_callnum_return is null and check_snum_return is null THEN  
       dbms_output.put_line(check_callnum_return);
       dbms_output.put_line(check_snum_return);

       --CHECKS TO SEE IF STUDENT IS EXIST IN ENROLLMENTS TABLE FOR DROP
       not_enrolled_dropme_return := not_enrolled_dropme(p_snum, p_callnum);
       IF not_enrolled_dropme_return is null THEN 
           dbms_output.put_line('student number ' || p_snum || ' is about to drop class number ' || p_callnum);
       
           --CHECKS TO SEE IF STUDENT HAS ALREADY BEEN GRADED
           already_graded_return := already_graded(p_snum, p_callnum);
           IF already_graded_return is null THEN 
               dbms_output.put_line('congrats you are not already graded for callnum ' 
                   || p_callnum || '. You have now been dropped from the course. (IN PROCEDURE)');
               drop_enrollments_return := drop_enrollments(p_snum, p_callnum);
               
               IF drop_enrollments_return is null THEN 
                   for eachstudent in (
                   select snum, callnum, time_waitlisted  
                   from waitlist  
                   where callnum = p_callnum 
                   order by time_waitlisted) LOOP

                       addme_proj(eachstudent.snum, eachstudent.callnum, p_out);
                       IF p_out is null THEN 
                       dbms_output.put_line('you have been added to ' ||p_callnum);
                       delete from waitlist where snum = eachstudent.snum and callnum = eachstudent.callnum;
                       exit;
                       end if;
                       END LOOP;
                   


               ELSE
                   dbms_output.put_line(drop_enrollments_return);
               END if;


           ELSE
               dbms_output.put_line(already_graded_return);
           END IF;

       ELSE 
           dbms_output.put_line(not_enrolled_dropme_return);
       END IF; 

   ELSE 
       dbms_output.put_line('ERROR: You may not drop (IN PROCEDURE)');
       dbms_output.put_line(check_callnum_return);
       dbms_output.put_line(check_snum_return);

   END IF; 


END; 

End Enroll;
/

