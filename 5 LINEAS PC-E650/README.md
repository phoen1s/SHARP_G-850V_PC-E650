# 5 Lineas Sharp PC-E650 y compatibles


Cómo activar el modo de 5 líneas en una Sharp PC-E650

A continuación explico los pasos para hacerlo fácilmente:  

	1.	Crear una unidad E: de 20 KB:  
	INIT "E:20K"   
	2.	Cargar el archivo 5INST.UU:  
	LOAD "COM:"   
	3.	Ejecutarlo:  
	RUN  
 
Si todo va bien, en la unidad E: se creará el archivo en código máquina 5LINST.  

	4.	Si tienes una SD en X:, puedes copiar ese archivo para uso futuro, por ejemplo:  
	COPY "E:5INST.UU" TO "X:5LINEAS.HBE"   
	5.	Ahora carga el archivo 5LINEAS.BAS:  
	LOAD "5LINEAS.BAS"   
	6.	Ejecuta:  
	RUN  

	•	Si no reservaste memoria para código máquina, pulsa N. 
 		La máquina se reiniciará y deberás volver a ejecutar con RUN.
   
	•	Si ya reservaste el área de código máquina, pulsa ENTER: 
 		el programa se copiará en memoria y se ejecutará.
 
	7.	Cambiar entre 4 y 5 líneas
 	INIT "SCRN:4"
 	INIT "SCRN:5"
