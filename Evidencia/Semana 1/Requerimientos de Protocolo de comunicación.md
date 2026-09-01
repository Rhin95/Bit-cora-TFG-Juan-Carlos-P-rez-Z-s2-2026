---
created: 2026-07-23T19:00
updated: 2026-08-26T18:47
---


Para la comunicación entre el módulo de comunicación embebido y el gateway se presume que este debe de soportar multiples sistemas embebidos funcionando simultáneamente. 

Los requerimientos identificados para la primera versión son:

- Se pretende que el receptor sea multicanal para el diseño final, a nivel comercial se manejan usualmente gateways de multiples canales pero 8 demoduladores que pueden funcionar de forma simultanea. La solución tiene que contemplar que para ser escalable debe de contemplar modelos de gateway comerciales.

- Tiene que tener un ACK (Acknowledgment) ya que es necesario confirmar que los datos fueron transferidos exitosamente al gateway    

- Se necesita desea asegurar que el 100% de los paquetes se envió y llegó con integridad adecuada, en caso de que alguno falle se tiene que reenviar. Para el contexto del proyecto se propone utilizar el 90% por ser la primera versión del protocolo.

- Como punto de entrada de la baliza al enlace de comunicación se define una FCinfo con SF12 (condiciones invariantes) en la que el gateway publica su disponibilidad de canales y la versión de configuración. Las balizas consultan este canal para saber la disponibilidad de los canales.

- El sistema embebido , luego de seleccionar un canal disponible aleatoriamente, debe escuchar ( CAD (Channel Activity Detection)) si el canal se encuentra libre de terceros o si en el periodo entre identificar el canal y escuchar el canal se ocupó. En caso de estar ocupado agrega dicho canal en una lista temporal de "canales intentados" y vuelve al canal FC para seleccionar otro canal.  La lista creada se reinicia luego de un tiempo de un periodo de minutos generado aleatoriamente entre un rango a definir.

- Para una segunda versión del protocolo se sugiere integrar un mecanismo de búsqueda estilo barrido en caso que la FCinfo se encuentre comprometida.
    
- Cada cierto tiempo debe monitorear la integridad del enlace, si este es suficientemente bueno (pendiente a definir "bueno") puede disminuir el SF o el Power para poder transferir datos más rápido, en caso que sea malo debe de modificar el SF o el Power para asegurarse que los datos lleguen bien. Para la primera versión se va a priorizar disminuir SF y se deja la apertura de poder disminuir Power pero ya que la potencia no es prioridad en este proyecto no se pretende generar dichos parámetros.
        
- La idea es que el estado base tanto del emisor como del receptor sea un FC fijo para cada canal con un SF determinado por ejemplo SF 12 y que de ahí se negocie el cambio dependiendo de la integridad del enlace.

- En el caso que un sistema embebido se le necesite cambiar los FC previamente programados tiene que existir algún mecanismo para cambiarlos desde la comunicación lora ya que no se pretende estar interviniendo físicamente a los mecanismos. Estos comando de reconfiguración se contemplan en la versión 2 del protocolo.

