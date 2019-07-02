# Postgis-Mapserver
Máquina virtual Debian 8.2 Amd64

PASSO 0 .INSTALAR DEPENDÊNCIAS PARA COMPILAÇÃO

	apt-get install build-essential make gcc linux-headers-$(uname -r) cmake

PASSO 1.INSTALAÇÃO DO POSTGRESQL

	apt-get install postgresql-9.4 postgresql-server-dev-9.4 libxml2-dev libproj-dev libjson0-dev libgeos-dev xsltproc docbook-xsl 	docbook-mathml libgdal-dev pgadmin3
	wget http://download.osgeo.org/postgis/source/postgis-2.0.7.tar.gz
	tar xfz postgis-2.0.7.tar.gz
	cd postgis-2.0.7
	./configure
	make
	make install
	ldconfig
	make comments-install

Configuração do postgres:

	sudo su
	su - postgres
	psql
	\password postgres
	Digite seu password
	\q
	nano /etc/postgresql/9.4/main/postgresql.conf

Aterar o seguinte parâmetro por:

	listen_addresses = '*'    
	nano /etc/postgresql/9.1/main/pg_hba.conf
Adicionar a seguinte linha com endereço da sua rede:

	host    all             postgres        192.168.0.0/24          md5
	/etc/init.d/postgresql restart

PASSO 2. INSTALACÃO DO POSTGIS

	wget http://download.osgeo.org/postgis/source/postgis-2.0.7.tar.gz
	tar xfz postgis-2.0.7.tar.gz
	cd postgis-2.0.7
	./configure
	make
	make install
	ln -sf /usr/share/postgresql-common/pg_wrapper /usr/local/bin/shp2pgsql
	ln -sf /usr/share/postgresql-common/pg_wrapper /usr/local/bin/pgsql2shp
	ln -sf /usr/share/postgresql-common/pg_wrapper /usr/local/bin/raster2pgsql

Criar base de dados:

	su - postgres
	createdb template_postgis
	createlang plpgsql template_postgis
	psql -d template_postgis -c "UPDATE pg_database SET datistemplate=true WHERE datname='template_postgis'"
	psql -d template_postgis -f /usr/share/postgresql/9.4/contrib/postgis-2.0/postgis.sql
	psql -d template_postgis -f /usr/share/postgresql/9.4/contrib/postgis-2.0/spatial_ref_sys.sql
	psql -d template_postgis -f /usr/share/postgresql/9.4/contrib/postgis-2.0/postgis_comments.sql
	psql -d template_postgis -f /usr/share/postgresql/9.4/contrib/postgis-2.0/rtpostgis.sql
	psql -d template_postgis -f /usr/share/postgresql/9.4/contrib/postgis-2.0/raster_comments.sql

PASSO 3. APACHE & PHP

Instalação do apache e módulos:

	apt-get install apache2
	a2enmod rewrite
	a2enmod actions
	a2enmod fcgid
	a2enmod suexec
	a2enmod include
	/etc/init.d/apache2 restart

intalação do php:

	apt-get install php5 php-pear php5-mysql php-apc php5-curl php5-pgsql php5-dev php5-xsl apache2-prefork-dev

PASSO 4. INSTALAÇÃO DE MAPSERVER 

Prequisitos de instalação:

	apt-get install swig libfreetype6-dev libcairo2-dev libapache2-mod-fcgid php5-cgi libfcgi-dev libapache2-mod-fastcgi apache2-suexec

Compilação e instalação:

	wget http://download.osgeo.org/mapserver/mapserver-7.0.0.tar.gz
	tar xzf mapserver-7.0.0.tar.gz
	cd mapserver-7.0.0/
	mkdir build
	cd build
	cmake -DCMAKE_INSTALL_PREFIX=/usr/lib -DCMAKE_PREFIX_PATH=/usr/local/pgsql/92:/usr/local:/usr -DWITH_CLIENT_WFS=ON -DWITH_CLIENT_WMS=ON -DWITH_CURL=ON -DWITH_SOS=ON -DWITH_PHP=ON -DWITH_PERL=0 -DWITH_RUBY=0 -DWITH_JAVA=0 -DWITH_CSHARP=0 -DWITH_PYTHON=0 -DWITH_MSSQL2008=0 -DWITH_FRIBIDI=0 -DWITH_HARFBUZZ=0 -DWITH_ORACLESPATIAL=0 -DWITH_SVGCAIRO=0 ../ > ../configure.out.txt


PASSO 5. ADICIONAR AO VIRTUAL HOST DO APACHE

nano /etc/sites-avaibles/mapserver.conf
	<VirtualHost *:80>
	    DocumentRoot "/var/www/"
	    ServerName www.example.com

	   ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
		<Directory "/usr/lib/cgi-bin">
		AllowOverride None
		Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
		Require all granted
		AddHandler fcgid-script .fcgi
		</Directory>

	</VirtualHost>
Habiitar host virtual:

	a2ensite mapserver.conf 

PASSO 6. CARREGAR DADOS NO POSTGIS
Dados do INEGI: https://www.inegi.org.mx/temas/mg/default.html#Descargas

	wget\ http://internet.contenidos.inegi.org.mx/contenidos/Productos/prod_serv/contenidos/espanol/bvinegi/productos/geografia/marc_geo/889463084105_s.zip889463084105_s.zip
	unzip 889463084105_s.zip
	shp2pgsql -s 4326 -W "latin1" areas_geoestadisticas_basicas.shp public.mge2015 | psql -h localhost -d geodb -U postgres

PASSO 7. CRIAR UM ARQUIVO MAPFILE

	mkdir /var/www/maps
	chown www-data:www-data /var/www/maps/
	nano /var/www/maps/mge.map
	 MAP
	 NAME "MGE"
	 SIZE 640 480
	 IMAGETYPE png
	 IMAGECOLOR 200 220 255
	 EXTENT -118.407650 14.532098 -86.710405 32.718654
	 CONFIG "MS_ERRORFILE" "/var/www/maps/ms_error.txt"
	 DEBUG 5
	 UNITS dd
	 PROJECTION
	 "init=epsg:4326"
	 END
	 WEB
	 #NAME "MARCO GEOESTADISTICO"
	 IMAGEPATH "/var/www/maps/"
	 IMAGEURL "/tmp/"
	 METADATA
	 "ows_enable_request" "*"
	 "wfs_enable_request" "*"
	 "wfs_title" "WFS Demo"
	 "wfs_onlineresource" "http://localhost/cgi-bin/mapserv.fcgi?map=/var/www/maps/mge.map&"
	 "wfs_srs" "EPSG:4326 EPSG:900913"
	 "ows_srs" "EPSG:4326 EPSG:900913"
	 "wfs_abstract" "Geotable"
	 END # metadata
	 END # web
	 LAYER
	 NAME 'geotable'
	 METADATA
	 "wms_title" "PostGIS WMS Layer"
	 "wms_srs" "epsg:900913 epsg:4326"
	 "wms_onlineresource" "http://localhost/cgi-bin/mapserv.fcgi?map=/var/www/maps/mge.map&"
	 "wms_enable_request" "GetCapabilities GetMap GetFeatureInfo"
	 "wms_extent" "-118.407650, 14.532098, -86.71.0405, 32.718654"
	 END
	 STATUS default
	 CONNECTIONTYPE postgis
	 CONNECTION "host=localhost user=postgres password=passwordpasso1 dbname=geodb"
	 DATA 'geom from mge2015 using unique gid using srid=4326'
	 TYPE polygon
	 CLASS
	 COLOR 255 50 50
	 OUTLINECOLOR 30 30 30
	 END
	 PROJECTION
	 "init=epsg:4326"
	 END
	 METADATA
	 "wfs_srs" "EPSG:4326"
	 END
	 END # End of layer
	END # End of map
	
Consulte:

	http://localhost/cgi-bin/mapserv.fcgi?map=/var/www/maps/mge.map&mode=map&layer=geotable



