drop table subcategory cascade constraints;
drop table category cascade constraints;
drop table product cascade constraints;
drop table storehouse cascade constraints;
drop table store_in cascade constraints;
drop table store_out cascade constraints;
drop table orders cascade constraints;
drop table order_unit cascade constraints;
drop table staff cascade constraints;
drop table client cascade constraints;
drop table role cascade constraints;
drop table commande cascade constraints;
drop table recevoir_produit cascade constraints;



/* category */
CREATE TABLE category(
  idc INTEGER PRIMARY KEY,
	category VARCHAR(15) NOT NULL
);

/* subcategory */
CREATE TABLE subcategory(
	ids INTEGER PRIMARY KEY,
	idc INTEGER REFERENCES category(idc),
	subcategory VARCHAR(15) NOT NULL	
);

/* product */
CREATE TABLE product(
	idp INTEGER PRIMARY KEY,
	name VARCHAR(20) NOT NULL,
	ids INTEGER REFERENCES subcategory(ids),
	price integer,
	but varchar(20) default 'vendre' ,
	CONSTRAINT CHECK_PIRCE CHECK (price>0),
	unique(but,name),
	check (but = 'acheter' or but = 'vendre')
);


/* role */
CREATE TABLE role(
	idr INTEGER PRIMARY KEY,
	role_name VARCHAR(10) NOT NULL
);

/* staff */
CREATE TABLE staff(
	ids INTEGER PRIMARY KEY,
	name VARCHAR(10) NOT NULL,
	age INTEGER,
	idr INTEGER REFERENCES role(idr),
	CONSTRAINT CHECK_STAFF_AGE CHECK (age > 0)
);

/* client */
CREATE TABLE client(
	idc INTEGER PRIMARY KEY,
	name VARCHAR(10) NOT NULL,
	age INTEGER,
	CONSTRAINT CHECK_CLIENT_AGE CHECK (age>0)
);

/* storehouse */
CREATE TABLE storehouse(
	idp INTEGER REFERENCES product(idp),
	quantity INTEGER NOT NULL,
	CONSTRAINT CHECK_STOREHOUSE CHECK(quantity > -1)
);


/* store_in */
CREATE TABLE store_in(
	ids_i INTEGER PRIMARY KEY,
	idp INTEGER REFERENCES product(idp),
	quantity INTEGER NOT NULL,
	t_date DATE,
	ids INTEGER REFERENCES staff(ids),
	CONSTRAINT CHECK_STOREIN CHECK (quantity > 0)
);

/* store_out */
CREATE TABLE store_out(
	ids_o INTEGER PRIMARY KEY,
	idp INTEGER REFERENCES product(idp),
	quantity INTEGER NOT NULL,
	t_date DATE,
	CONSTRAINT CHECK_QUANTITY CHECK (quantity>0)
);

/* order */
CREATE TABLE orders(
	ido INTEGER PRIMARY KEY,
	idc INTEGER REFERENCES client(idc),
	t_date DATE
);

/* order_unit */
CREATE TABLE order_unit(
	ido_u INTEGER PRIMARY KEY,
	idp INTEGER REFERENCES product(idp),
	quantity INTEGER NOT NULL,
	ido INTEGER REFERENCES orders(ido),
	CONSTRAINT CHECK_ORDERUNIT CHECK (quantity>0)
);


---------table commande pour les client faits les commande de produit
create  table commande (
ic integer primary key,
client varchar(40) not null,
produit integer references product(idp),
quntite integer not null,
cleTemp integer not null
);
----------table pour enrgistrer les information de produit qu'on a achete--------
create table recevoir_produit
(
irp integer primary key,
vendeur varchar(20) not null,
numcommande integer not null,
ids_i integer references store_in(ids_i)
);

/*------------------------------------------------*/
/* triggers */

/* trigger in table store_in, update table        */
/* storehouse automatically                       */
CREATE OR REPLACE TRIGGER store_in_trigger
AFTER INSERT ON store_in
FOR EACH ROW	
BEGIN
	UPDATE storehouse SET quantity = quantity + :new.quantity WHERE idp = :new.idp;
END;
/

/* trigger in table store_out, update table       */
/* storehouse automatically                       */
CREATE OR REPLACE TRIGGER store_out_trigger
BEFORE INSERT ON store_out
FOR EACH ROW	
BEGIN
	UPDATE storehouse SET quantity = quantity - :new.quantity WHERE idp = :new.idp;
END;
/









---------------------------les sequance--------
DROP SEQUENCE seq_category;
DROP SEQUENCE seq_subcategory;
DROP SEQUENCE seq_product;
DROP SEQUENCE seq_storehouse; 
DROP SEQUENCE seq_store_in; 
DROP SEQUENCE seq_store_out; 
DROP SEQUENCE seq_orders; 
DROP SEQUENCE seq_order_unit; 
DROP SEQUENCE seq_staff; 
DROP SEQUENCE seq_client; 
DROP SEQUENCE seq_role; 
drop sequence seq_commande;
drop sequence seq_recevoir_produit;

CREATE SEQUENCE seq_category START WITH 1;
CREATE SEQUENCE seq_subcategory START WITH 1;
CREATE SEQUENCE seq_product START WITH 1;
create sequence seq_storehouse start with 1;
CREATE SEQUENCE seq_store_in START WITH 1;
CREATE SEQUENCE seq_store_out START WITH 1;
CREATE SEQUENCE seq_orders START WITH 1;
CREATE SEQUENCE seq_order_unit START WITH 1;
CREATE SEQUENCE seq_staff START WITH 1;
CREATE SEQUENCE seq_client START WITH 1;
CREATE SEQUENCE seq_role START WITH 1;
create sequence seq_commande start with 1;
create sequence seq_recevoir_produit start with 1;
-----------------------les vues pour encapsule
create or replace view vue_category as select * from category;

create or replace view vue_client as select * from client;

create or replace view vue_order_unit as select * from order_unit;

create or replace view vue_orders as select * from orders;

create or replace view vue_product as select * from product;

create or replace view vue_role as select * from role;

create or replace view vue_staff as select * from staff;

create or replace view vue_store_in as select * from store_in;

create or replace view vue_store_out as select * from store_out;

create or replace view vue_storehouse as select * from storehouse;

create or replace view vue_subcategory as select * from subcategory;

create or replace view vue_commande as select * from commande;

create or replace view vue_recevoir_produit as select * from recevoir_produit;

------------vue info pour la fonction information
create or replace view vue_info_product as 
select p.idp , p.name,sub.subcategory,p.price,s.quantity
from storehouse s,product p,subcategory sub 
where s.idp = p.idp and p.ids = sub.ids and p.but = 'vendre';
---------------fonction pour consulter vendre et recoir les produit  ------------------------------
create or replace procedure informationproduit(nom varchar2)
is 
cursor c is 
select * 
from dwan_a.vue_info_product p
where nom = p.name;
test integer;
begin
	test :=0;
	for x in c loop
		dbms_output.put_line('NAME : '|| x.name );
		dbms_output.put_line('SUBCATEGORY :'||x.subcategory);
		dbms_output.put_line('PRICE : '|| x.price);
		dbms_output.put_line('QUANTITY : '||x.quantity);
		test := 1;
		end loop ;
		if(test=0)then
		dbms_output.put_line('no product or name incorrect');
		end if ;
		

end;
/

---------fonction permet tester si c'est un boncommande ou non
-----------------si oui revoie la identifiant de produit que le client demend
------------si non revoie -1
create or replace function boncommande(p varchar2,q integer )return integer is 
x integer;
num integer;
begin
x:=-1;
select vue.idp into num
from dwan_a.vue_info_product vue
where p = vue.name and (q < vue.quantity or q = vue.quantity);
return num;
exception
when  others then return x;
end;
/
----------------function permet tester le produit est dans la table product ou non
-------------utilise dans fonction recevoircommande-----
create or replace function existproduit(produit varchar,but varchar)return integer
is
idp integer;
begin
idp := -1;
select p.idp into idp
from dwan_a.vue_product p
where p.name = produit and p.but = but;
return idp;
exception when others then return idp;
end;
/
create or replace procedure recevoircommande(produit varchar,quantite integer,numcommande integer)is
num_idp integer;num_ids_i integer;
begin
num_idp := existproduit(produit,'acheter');
if num_idp = -1 then
num_idp := seq_product.nextval;
insert into product(idp,name,but) values(num_idp,produit,'acheter');
insert into storehouse values(num_idp,0);
end if;
num_ids_i := seq_store_in.nextval;
insert into store_in(ids_i,idp,quantity) values(num_ids_i,num_idp,quantite);
insert into recevoir_produit values(seq_recevoir_produit.nextval,user,numcommande,num_ids_i);

---------------je n'ai pas ajouter les data()--------
end;
/


create or replace procedure livrerproduit(ache varchar2,numcommande integer)is
text varchar2(400);
pro varchar(40);
quan integer;
identifiant integer;
begin
select p.name,c.quntite,p.idp  into pro,quan,identifiant
from dwan_a.vue_commande c,dwan_a.vue_product p
where c.ic = numcommande and p.idp = c.produit;
insert into dwan_a.store_out(ids_o,idp,quantity) values(seq_store_out.nextval,identifiant,quan);
text := ' begin '||ache||'.'||'recevoircommande( :1 , :2 , :3);   end;';
execute immediate text using pro,quan,numcommande ;
dbms_output.put_line('livrerproduit');
end;
/

create or replace procedure commandeproduit(produit in varchar2 , quantite in integer , clefTemp in integer)is
test integer; 
num integer;
testbanqueCCI integer;
prix integer;
begin
num :=seq_commande.nextval;
test := dwan_a.boncommande(produit,quantite);

if(test = -1)
then dbms_output.put_line('nom incorrect ou quantite deborder ');
else 
select p.price into prix
from dwan_a.vue_product p
where p.idp = test;
testbanqueCCI :=dmaison_a.facturercommande('dwan_a',user,clefTemp,prix*quantite,num);
if (testbanqueCCI = 0) then
dbms_output.put_line('probleme sur banque ou CCI');
else
insert into commande values(num,user,test,quantite,clefTemp);
dwan_a.livrerproduit(user,num);
end if;
end if ;
end;
/
 



---------- il faut donner les droit 
grant execute on commandeproduit to public;
grant execute on recevoircommande to public;
grant execute on informationproduit to public;

/* records in table category */
INSERT INTO category VALUES (seq_category.nextval, 'book');
INSERT INTO category VALUES (seq_category.nextval, 'clothes');
INSERT INTO category VALUES (seq_category.nextval, 'food');

/* records in table subcategory */
/* records in table subcategory which belongs to category 'book' */
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 2, 'literature');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 2, 'textbook');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 2, 'history');
/* records in table subcategory which belongs to category 'clothes' */
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 3, 'T-shirt');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 3, 'sock');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 3, 'hat');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 3, 'tie');
/* records in table subcategory which belongs to category 'food' */
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 4, 'drink');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 4, 'snack');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 4, 'meat');
INSERT INTO subcategory VALUES (seq_subcategory.nextval, 4, 'vegetable');

/* records in table product */
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Notre Dame a Paris', 2, 10);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Don Quijote', 2, 15);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'English lang', 3, 12);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Francais langue', 3, 13);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'French History', 4, 20);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Chinese History', 4, 20);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Adi_123', 5, 10);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Kappa_k25', 5, 100);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Nike_c2012', 6, 3);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'NY_YANKEE_11', 7, 10);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Crocodile_2011', 8, 12);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Milk', 9, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Cocacole', 9, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Spring', 9, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, '7up', 9, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Fromage Paris', 10, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Veau', 11, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Chicken', 11, 2);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Parsley', 12, 1);
INSERT INTO product(idp,name,ids,price) VALUES (seq_product.nextval, 'Bean', 12, 2);

/* records in table storehouse */
INSERT INTO storehouse VALUES (2, 10);
INSERT INTO storehouse VALUES (3, 10);
INSERT INTO storehouse VALUES (4, 10);
INSERT INTO storehouse VALUES (5, 10);
INSERT INTO storehouse VALUES (6, 10);
INSERT INTO storehouse VALUES (7, 10);
INSERT INTO storehouse VALUES (8, 10);
INSERT INTO storehouse VALUES (9, 10);
INSERT INTO storehouse VALUES (10, 10);
INSERT INTO storehouse VALUES (11, 10);
INSERT INTO storehouse VALUES (12, 10);
INSERT INTO storehouse VALUES (13, 10);
INSERT INTO storehouse VALUES (14, 10);
INSERT INTO storehouse VALUES (15, 10);
INSERT INTO storehouse VALUES (16, 10);
INSERT INTO storehouse VALUES (17, 10);
INSERT INTO storehouse VALUES (18, 10);
INSERT INTO storehouse VALUES (19, 10);
INSERT INTO storehouse VALUES (20, 10);
INSERT INTO storehouse VALUES (21, 10);

/* records in table role */
INSERT INTO role VALUES (seq_role.nextval, 'manager');
INSERT INTO role VALUES (seq_role.nextval, 'secretary');
INSERT INTO role VALUES (seq_role.nextval, 'employe');

/* records in table staff */
INSERT INTO staff VALUES (seq_staff.nextval, 'James', 30, 2);
INSERT INTO staff VALUES (seq_staff.nextval, 'Adams', 22, 4);
INSERT INTO staff VALUES (seq_staff.nextval, 'Anne', 23, 3);
INSERT INTO staff VALUES (seq_staff.nextval, 'Odi', 25, 4);
INSERT INTO staff VALUES (seq_staff.nextval, 'Tank', 32, 4);

/* records in table client */
INSERT INTO client VALUES (seq_client.nextval, 'Bin', 35);
INSERT INTO client VALUES (seq_client.nextval, 'Cheng', 42);
INSERT INTO client VALUES (seq_client.nextval, 'Cris', 55);

/* records in table store_in */
INSERT INTO store_in VALUES (seq_store_in.nextval, 2, 3, to_date('2012-09-17', 'yyyy-mm-dd'), 3);
INSERT INTO store_in VALUES (seq_store_in.nextval, 4, 4, to_date('2012-09-16', 'yyyy-mm-dd'), 5);
INSERT INTO store_in VALUES (seq_store_in.nextval, 6, 4, to_date('2012-09-15', 'yyyy-mm-dd'), 5);
INSERT INTO store_in VALUES (seq_store_in.nextval, 7, 5, to_date('2012-09-14', 'yyyy-mm-dd'), 5);
INSERT INTO store_in VALUES (seq_store_in.nextval, 9, 2, to_date('2012-09-13', 'yyyy-mm-dd'), 5);
INSERT INTO store_in VALUES (seq_store_in.nextval, 13, 6, to_date('2012-09-12', 'yyyy-mm-dd'), 3);

/* records in table store_out */
INSERT INTO store_out VALUES (seq_store_out.nextval, 3, 1, to_date('2012-09-17', 'yyyy-mm-dd'));
INSERT INTO store_out VALUES (seq_store_out.nextval, 5 , 1, to_date('2012-09-16', 'yyyy-mm-dd'));
INSERT INTO store_out VALUES (seq_store_out.nextval, 8, 2, to_date('2012-09-15', 'yyyy-mm-dd'));
INSERT INTO store_out VALUES (seq_store_out.nextval, 9, 1, to_date('2012-09-13', 'yyyy-mm-dd'));
INSERT INTO store_out VALUES (seq_store_out.nextval, 12, 1, to_date('2012-09-12', 'yyyy-mm-dd'));

/* records in table orders */
INSERT INTO orders VALUES (seq_orders.nextval, 2, to_date('2012-09-14', 'yyyy-mm-dd'));
INSERT INTO orders VALUES (seq_orders.nextval, 3, to_date('2012-09-13', 'yyyy-mm-dd'));
INSERT INTO orders VALUES (seq_orders.nextval, 4, to_date('2012-09-12', 'yyyy-mm-dd'));

/* records in table order_unit */
/* records in table order_unit composing the 1st order */
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 2, 1, 2);
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 3, 1, 2);
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 4, 1, 2);
/* records in table order_unit composing the 2nd order */
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 5, 1, 3);
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 7, 1, 3);
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 8, 1, 3);
/* records in table order_unit composing the 3rd order */
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 2, 1, 4);
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 4, 1, 4);
INSERT INTO order_unit VALUES (seq_order_unit.nextval, 21, 1, 4);


