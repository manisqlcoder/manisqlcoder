CREATE OR REPLACE package body TTSBANK.pkg_bankproject as

procedure sp_master_details( p_bank_details out  pkg_bankproject.ref_common , p_product_details out  pkg_bankproject.ref_common) as
begin
open p_bank_details for select bank_id , bank_code , bank_location from ttsbank_banks;
open p_product_details for select product_id, product_name, product_code from ttsbank_products;
end;

procedure sp_ttsbank_newaccount
(
p_bank_id in number,
p_product_id in number,
p_customer_id in number,
p_customer_name in varchar2,
p_customer_phno in number,
p_customer_mail in varchar2,
p_aadhar_no in number,
p_pan_no in varchar2,
p_password in varchar2,
p_trans_amount in number,
p_trans_type in varchar2,
p_trans_for in varchar2,
p_trans_mode in varchar2,
p_action in number,
p_msg out varchar2
)as
l_cusid number;
l_sub_product_id number;
l_cus_product_id number;
l_account_no number;
l_cnt number;
l_query varchar2(2000);
begin
if p_action = 1 then
insert into ttsbank_customers(customer_id, customer_name, customer_phno, customer_mail, aadhar_no, pan_no, login_password) values(sqcusid.nextval , p_customer_name , p_customer_phno , p_customer_mail , p_aadhar_no , p_pan_no ,p_password ) returning customer_id into l_cusid;

select sub_product_id into l_sub_product_id
from ttsbank_sub_products where product_id = p_product_id and rownum =1;

insert into ttsbank_cus_products(cus_product_id, sub_product_id, customer_id, account_no, account_open_on, status, available_balance) values(sqcusprodid.nextval , l_sub_product_id , l_cusid , sqaccno.nextval , sysdate , 'Active' , p_trans_amount) returning cus_product_id , account_no into l_cus_product_id ,l_account_no;

insert into ttsbank_cus_transactions(cus_trans_id, cus_product_id, trans_amount, trans_type, trans_on, trans_for, trans_mode) values(sqcustransid.nextval , l_cus_product_id , p_trans_amount , p_trans_type , sysdate , p_trans_for , p_trans_mode);

p_msg := 'Account Created and your A/c No is '|| l_account_no;

elsif p_action = 2 then

select count(1) into l_cnt from dual where p_password
in ( select customer_password
from ttsbank_password_track
where password_changed_on > add_months(sysdate,-3) and  customer_id = p_customer_id and rownum <= 3
union all
select login_password from ttsbank_customers where customer_id = p_customer_id);

if l_cnt = 0 then
update ttsbank_customers set login_password = p_password where customer_id = p_customer_id;
p_msg := 'Password Changed';
else
raise_application_error(-20000 , 'Same Password Kept Already');
end if;

elsif p_action = 3 then 

if p_customer_name is not null and p_customer_phno is not null  then

l_query := 'customer_name = '||p_customer_name  ||', customer_phno = '||p_customer_phno;
p_msg := 'Both name and phno updated';
elsif p_customer_name is not null and p_customer_phno is null  then

l_query := 'customer_name = '||p_customer_name;
p_msg := 'Name updated';
elsif p_customer_name is null and p_customer_phno is not null  then

l_query := 'customer_phno = '||p_customer_phno;
p_msg := 'Phno updated';
end if;

execute immediate ' update ttsbank_customers set '||l_query||' where customer_id = '||p_customer_id;

--dbms_output.put_line(' update ttsbank_customers set '||l_query||' where customer_id = '||p_customer_id);

end if;

end;

procedure sp_ttsbank_cus_transactions(
p_customer_id in number , 
p_product_id in number , 
p_trans_amount in number , 
p_trans_type in varchar2 , 
p_trans_for in varchar2 , 
p_trans_mode in varchar2, 
p_msg out varchar2 )as
l_test number;
l_sum_amount  number;
l_withdraw_limit number;
e_limitexceeded exception;
l_cnt number;
begin

if p_trans_type = 'd' then

select count(1) into l_cnt from ttsbank_cus_transactions a, ttsbank_cus_products b
where a.cus_product_id = b.cus_product_id and b.customer_id = p_customer_id and trans_type = 'd';

if l_cnt <> 0 then
select  sum(trans_amount) , withdraw_limit into  l_sum_amount , l_withdraw_limit 
from ttsbank_cus_transactions a, ttsbank_cus_products b , ttsbank_sub_products c
where a.cus_product_id = b.cus_product_id and b.sub_product_id = c.sub_product_id
and b.customer_id = p_customer_id and trunc(trans_on) = trunc(sysdate)
and trans_type = 'd' group by withdraw_limit;
end if;

if (l_sum_amount+p_trans_amount) > l_withdraw_limit then
raise e_limitexceeded;
end if;

end if;

insert into ttsbank_cus_transactions(cus_trans_id, cus_product_id, trans_amount, trans_type, trans_on, trans_for, trans_mode) values(sqcustransid.nextval , p_product_id , p_trans_amount , p_trans_type , sysdate , p_trans_for , p_trans_mode);

update ttsbank_cus_products 
set available_balance = case when p_trans_type = 'c' then available_balance+p_trans_amount when p_trans_type = 'd' then available_balance-p_trans_amount  end where customer_id = p_customer_id;

p_msg := 'Transaction Success';

exception
when e_limitexceeded then
p_msg := 'Daily Limit Exceeded';

end;

end;
/
