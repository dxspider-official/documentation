# Guía de actualización de DXSpider a la última build de Mojo

Esta guía está destinada a SYSOPs que ya disponen de una instalación de DXSpider. Se puede realizar la actualización manualmente o utilizar el script automático.

Los ejemplos usan `/spider` como ruta de acceso a la instalación y `sysop` como usuario habitual. Compruebe siempre los valores reales de su equipo.

## Método 1. Actualización manual

### 1. Instalar los paquetes necesarios

En Debian, Ubuntu, Raspberry Pi OS y distribuciones derivadas, acceda como administrador:

```bash
sudo -i
apt-get update
apt-get install perl git rsync curl iproute2 gzip procps build-essential cpanminus libssl-dev zlib1g-dev
```

Instale los paquetes Perl disponibles. El siguiente bucle permite continuar si algún paquete no existe en su distribución (ejecútelo en bash):

```bash
for pkg in libtimedate-perl libnet-telnet-perl libcurses-perl libdigest-sha-perl \
    libdata-dumper-simple-perl libjson-perl libmojolicious-perl \
    libdata-structure-util-perl libmath-round-perl libev-perl \
    libjson-xs-perl libnet-cidr-lite-perl libdbi-perl libdbd-sqlite3-perl \
    libhtml-parser-perl libio-socket-inet6-perl libarchive-zip-perl \
    libmailtools-perl libwww-perl libio-pty-perl libauthen-sasl-perl \
    libnet-smtp-ssl-perl libemail-sender-perl; do
    apt-get install -y "$pkg" || echo "Revisar paquete: $pkg"
done
```

Si algún módulo no está disponible mediante APT, utilice `cpanm`. Este bloque comprueba todos los módulos requeridos por el actualizador e instala únicamente los que faltan:

```bash
for module in EV Mojolicious JSON JSON::XS Data::Structure::Util Math::Round \
    Net::CIDR::Lite DBI DBD::SQLite Date::Parse HTML::Parser \
    IO::Socket::INET6 Archive::Zip Curses Mail::Internet Digest::SHA \
    LWP::UserAgent LWP::Parallel::UserAgent DB_File IO::Pty \
    Authen::SASL Net::SMTP::SSL Email::Sender; do
    perl -M"$module" -e 1 2>/dev/null || cpanm "$module" || break
done
```

Compruebe que dispone de Perl 5.16.1 o posterior y Mojolicious 7.26 o posterior:

```bash
perl -e 'die "Perl demasiado antiguo\n" if $] < 5.016001'
perl -MMojolicious -e 'Mojolicious->VERSION(7.26)'
```

Si alguna instalación o comprobación falla, no continúe hasta resolverla.

### 2. Configurar la base de datos de usuarios

Identifique el usuario propietario y la ruta real:

```bash
readlink -f /spider
stat -c '%U:%G' /spider/perl/cluster.pl
```

Cambie al usuario que ejecuta DXSpider (normalmente `sysop`; sustituya el nombre si es diferente):

```bash
su - sysop
cd /spider
pwd -P
nano /spider/local/DXVars.pm
```

Añada la siguiente línea antes del `1;` final, únicamente si `$userdsn` no está ya definido:

```perl
our $userdsn = "dbi:SQLite:dbname=$root/local_data/dxusers.db";
```

Guarde el fichero. No elimine el `1;` final ni duplique la variable.

### 3. Actualizar las tareas programadas

```bash
nano /spider/local_cmd/crontab
```

Sustituya las tareas equivalentes existentes por las siguientes, cuando proceda. No duplique tareas ni elimine otras tareas propias del nodo, como las conexiones a otros clústeres.

Seleccione únicamente uno de los dos bloques BadIP según la build instalada. Elimine o comente el otro bloque. Si dispone de una IP pública estática, la tarea programada `set/external_ip` es opcional.

```perl
# Update dynamic public IP
0,30 * * * * run_cmd('set/external_ip')

# Automatic compression debug and log files
10 0 * * * spawn("$main::root/perl/compression.pl")

# AutoUpdate badspotter badnode
10 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badspotter')
11 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badnode')
12 * * * * run_cmd('load/badnode')
12 * * * * run_cmd('load/badspotter')

# AutoUpdate BadIP (only if build < 935)
33 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.torexit')
34 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.torrelay')
35 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.global')
36 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.curated')
40 * * * * run_cmd('load/badip')

# AutoUpdate BadIP (only if build > 935)
33 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip*')
40 * * * * run_cmd('load/badip')

# AutoUpdate USDB
0 1 * * * run_cmd('download https://ftp.w1nr.net/usdbraw.gz')
2 1 * * * spawn('/spider/perl/create_usdb.pl /spider/local_data/usdbraw.gz')
6 1 * * * run_cmd('load/usdb')
9 1 * * * spawn('rm -f /spider/local_data/usdbraw.gz')

# AutoUpdate Keps
0 2 * * * run_cmd('download http://www.amsat.org/amsat/ftp/keps/current/nasabare.txt')
2 2 * * * spawn('/spider/perl/convkeps.pl -p /spider/local_data/nasabare.txt')
6 2 * * * run_cmd('load/keps')
9 2 * * * spawn('rm -f /spider/local_data/nasabare.txt')

# AutoUpdate CTY.DAT
0 3 * * * run_cmd('download http://www.country-files.com/cty/cty.dat')
2 3 * * * run_cmd('download http://www.country-files.com/cty/wpxloc.raw')
6 3 * * * spawn("/spider/perl/create_prefix.pl")
9 3 * * * run_cmd("load/prefix")
```

### 4. Actualizar el fichero startup

```bash
nano /spider/scripts/startup
```

Elimine, si existen, las líneas que comiencen por las siguientes expresiones, independientemente del valor que tengan asignado:

```text
set/var $main::maxconnect_user
set/var $main::localhost_alias_ipv4
set/var $main::localhost_alias_ipv6
set/var @main::localhost_names
```

Conserve el resto del fichero.

### 5. Actualizar a la última build de Mojo

Ejecute Git como propietario de la instalación, no como root (salvo que root sea realmente el usuario de DXSpider). Sitúese en el directorio de DXSpider, no en el directorio personal:

```bash
cd /spider
pwd -P
git rev-parse --show-toplevel
git remote -v
```

Compruebe que la raíz del repositorio coincide con la instalación real. Si no es un repositorio Git, utilice el método automático.

Descargue la rama `mojo` del repositorio oficial de Dirk:

```bash
git fetch git://scm.dxcluster.org/spider refs/heads/mojo
```

**Solo si falla**, utilice el repositorio alternativo:

```bash
git fetch https://github.com/EA3CV/dxspider.git refs/heads/mojo
```

Si ambos fallan, no continúe. Después de una descarga correcta:

```bash
git log -1 --oneline FETCH_HEAD
```

Antes de sustituir el código, detenga el nodo como se indica en el apartado 6 y haga una copia completa de la instalación con el nodo parado, fuera del árbol de DXSpider. Compruebe que dispone de espacio suficiente y que la copia incluye los datos y la configuración local.

**ATENCIÓN:** los siguientes comandos descartan cambios locales en los archivos versionados y sustituyen la rama local `mojo` por la revisión descargada. No utilice `git clean`, ya que podría eliminar archivos locales necesarios. Si tiene configuración o datos locales versionados, consérvelos en la copia y revise su restauración antes de arrancar.

```bash
cd /spider
git reset --hard FETCH_HEAD && git checkout -f -B mojo FETCH_HEAD
```

Verifique el resultado:

```bash
git branch --show-current
git log -1 --oneline
git status --short
```

La rama debe ser `mojo`. No continúe si Git devuelve un error. Los archivos versionados deben corresponder a la revisión descargada; los archivos locales no versionados se conservan salvo los que impidan escribir archivos versionados. Compruebe que las modificaciones de los apartados 2, 3 y 4 siguen presentes.

### 6. Reiniciar y verificar el nodo

#### Detener el nodo (antes de aplicar el cambio de código del apartado 5)

Si DXSpider funciona como servicio systemd, desde una cuenta con permisos de administrador:

```bash
sudo systemctl stop dxspider.service
sudo systemctl status dxspider.service
```

Si funciona manualmente, entre en la consola de DXSpider con privilegios de SYSOP y ejecute:

```text
shut
```

No confunda `shut` con un comando del shell de Linux. Compruebe que el nodo se ha detenido antes de continuar.

#### Comprobar el arranque manual

Una vez actualizado, ejecute como usuario de DXSpider:

```bash
cd /spider/perl
/spider/perl/cluster.pl
```

Observe los mensajes de arranque. Si aparecen errores, resuélvalos antes de iniciar el servicio. Si el arranque es correcto, abra otra terminal, acceda a la consola (habitualmente `/spider/perl/console.pl`) y ejecute `shut` para detener el proceso de prueba.

#### Arranque definitivo

Si utiliza systemd, desde una cuenta con permisos administrativos:

```bash
sudo systemctl start dxspider.service
sudo systemctl status dxspider.service
```

También puede utilizar `sudo service dxspider start` si su instalación admite ese comando.

Si utiliza el arranque manual, como usuario de DXSpider:

```bash
cd /spider/perl
/spider/perl/cluster.pl
```

Acceda a la consola y compruebe la versión y build con `show/version`.

## Método 2. Actualización automática

Como alternativa al método manual, realice únicamente los apartados 3 y 4 anteriores y utilice el actualizador de EA3CV, que instala dependencias, prepara una copia de seguridad y actualiza a la rama `mojo`, intentando primero el repositorio de Dirk y después GitHub.

Repositorio: https://github.com/EA3CV/dxspider_info

Antes de ejecutarlo, averigüe la ruta real de la instalación:

```bash
readlink -f /spider
```

Si el nodo funciona manualmente, deténgalo mediante `shut` desde la consola. Si utiliza systemd y está activo, el actualizador gestiona su parada.

Acceda como root y descargue DXSpider Manager en un directorio de trabajo nuevo:

```bash
sudo -i
cd /root
mkdir dxspider-manager
cd dxspider-manager
apt-get install wget
wget -O dxspider-manager.pl https://raw.githubusercontent.com/EA3CV/dxspider_info/main/dxspider-manager.pl
chmod a+x dxspider-manager.pl
./dxspider-manager.pl
```

Si el directorio de trabajo ya existe, utilice otro nombre para evitar mezclar descargas antiguas. Si ya es root, omita `sudo -i`.

Cuando el script solicite el directorio de instalación de DXSpider, introduzca la ruta real obtenida anteriormente (por ejemplo `/home/sysop/spider`). No indique `/home/sysop` ni el directorio donde ha descargado el actualizador.

Seleccione la opción de actualización de una instalación existente y siga las instrucciones que aparecen en pantalla. Al finalizar, compruebe el arranque del nodo y su versión. No seleccione una instalación nueva.
