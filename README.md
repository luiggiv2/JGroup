# Aplicación utilizando JGroups.

Este código implementa un chat sencillo utilizando JGroups, una biblioteca Java para comunicación en grupos/clusters de nodos en redes distribuidas. Se configura una instancia de **clustering** o **multicasting** para comunicación entre nodos (es decir, entre diferentes instancias de una aplicación o servicio que deben mantenerse sincronizados). Esto es común en sistemas distribuidos, donde cada nodo necesita recibir mensajes de otros nodos en tiempo real.

```java
public class SimpleChat implements Receiver {

    JChannel channel;
    String user_name = System.getProperty("user.name", "n/a");
    final List<String> state = new LinkedList<>();

    private void start() throws Exception {
        channel = new JChannel().setReceiver(this);
        channel.connect("ChatCluster");
        channel.getState(null, 10000);
        eventLoop();
        channel.close();
    }

    private void eventLoop() {
        BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
        while (true) {
            try {
                System.out.print("> ");
                System.out.flush();
                String line = in.readLine().toLowerCase();
                if (line.startsWith("quit") || line.startsWith("exit")) {
                    break;
                }
                line = "[" + user_name + "] " + line;
                Message msg = new ObjectMessage(null, line);
                channel.send(msg);
            } catch (Exception e) {
            }
        }
    }

    @Override
    public void receive(Message msg) {
        String line = msg.getSrc() + ": " + msg.getObject();
        System.out.println(line);
        synchronized (state) {
            state.add(line);
        }
    }

    public static void main(String[] args) throws Exception {
        new SimpleChat().start();
    }

    @Override
    public void getState(OutputStream output) throws Exception {
        synchronized (state) {
            Util.objectToStream(state, new DataOutputStream(output));
        }
    }

    @Override
    public void setState(InputStream input) throws Exception {
        List<String> list;
        list = (List<String>) Util.objectFromStream(new DataInputStream(input));
        synchronized (state) {
            state.clear();
            state.addAll(list);
        }
        System.out.println(list.size() + " messages in chat history):");
        list.forEach(System.out::println);
    }

}
```

## Aquí hay una explicación detallada de cómo funciona cada parte:
**1. Clase `SimpleChat` y su implementación de `Receiver`**
   - La clase `SimpleChat` implementa la interfaz `Receiver` de **JGroups**, lo cual le permite recibir mensajes y manejar la sincronización de estado entre nodos del cluster (en este caso, las instancias de chat). Al implementar `Receiver`, el programa gestiona la recepción de mensajes y la sincronización del historial de mensajes entre los miembros del chat.
     
**2. Variables de clase**
   - `JChannel channel` es el canal de comunicación que conecta a este nodo con otros en el cluster de chat. Los mensajes se envían y reciben a través de este canal.
   - `JString user_name` es el nombre del usuario actual, obtenido del sistema. Se utiliza para identificar al usuario en los mensajes.
   - `List<String> state` mantiene un historial de mensajes en el chat.

**3. Método `start()`**
El método `start()` inicializa y configura el canal de comunicación de JGroups. Aquí están los pasos clave:
  - **Inicialización** se crea un canal (`channel = new JChannel()`) y se establece `this` como receptor de mensajes (`setReceiver(this)`).
  - **Conexión al cluster** el canal se conecta al grupo llamado `"ChatCluster"` usando `channel.connect("ChatCluster")`.
  - **Sincronización del estado** se llama a `channel.getState(null, 10000)`, solicitando el historial de mensajes existentes en el cluster. Este proceso se completa en `setState`.
  - **Loop de eventos** llama a `eventLoop()`, un método que gestiona la entrada de mensajes del usuario.
  - **Cierre del canal** cuando el usuario termina la sesión de chat, el canal se cierra con `channel.close()`.

**4. Método `eventLoop()`**
Este método implementa el loop de entrada del usuario para enviar mensajes.
  - Usa `BufferedReader` para leer la entrada de la consola.
  - Los mensajes se formatean como `"[user_name] mensaje"` y se envían a través del canal usando `channel.send(msg)`.
  - Los mensajes se envían como instancias de `ObjectMessage`, que permiten transportar objetos arbitrarios.

**5. Método `receive(Message msg)`**
Este método implementa la recepción de mensajes de otros nodos del cluster.
  - **getState** serializa y envía el historial de mensajes actual (`state`) cuando un nuevo nodo se une al cluster.
  - **setState** recibe el estado del historial de mensajes al unirse al cluster, reemplazando el historial local `state` con el historial compartido.

**6. Método `main`**
Es el punto de entrada de la aplicación. `Simplemente` crea una instancia de SimpleChat y llama al método `start()`.

**7. Métodos `getState` y `setState`**
Estos métodos gestionan la sincronización del historial de mensajes en los nodos.
  - **getState** serializa y envía el historial de mensajes actual (`state`) cuando un nuevo nodo se une al cluster.
  - **setState** recibe el estado del historial de mensajes al unirse al cluster, reemplazando el historial local `state` con el historial compartido.

**Ejecución en conjunto**
  - Cada vez que se inicia una instancia de `SimpleChat`, el programa se conecta a `ChatCluster`, sincroniza el historial de mensajes y permite la comunicación entre instancias.
  - Los mensajes se envían y reciben en tiempo real, permitiendo un chat sencillo entre múltiples usuarios.

Este código proporciona una base para una aplicación de chat distribuida, fácil de ampliar para otros escenarios donde múltiples instancias necesitan comunicarse y sincronizarse en tiempo real.

Si ejecutamos el programa en dos ocasiones se nos abrirá dos ventanas para poder implementar el chat.

**Aquí observamos que el usuario `LV2-64046` envía un mensaje.**
<img width="717" alt="Captura de pantalla 2024-11-01 a la(s) 11 32 44 p m" src="https://github.com/user-attachments/assets/3de97343-d619-4bc8-ae9a-622d091ca2ca">

**Aquí observamos que el usuario `LV2-56871` recibe el mensaje enviado por el usuario `LV2-64046`.**
<img width="747" alt="Captura de pantalla 2024-11-01 a la(s) 11 33 40 p m" src="https://github.com/user-attachments/assets/746d3aff-094f-431e-aa07-09965e0c83cc">

**Si ejecutamos la aplicacion una tercera vez observamos que el usuario `LV2-57471` recibe los mensaje sincronizados de los `LV2-64046` y `LV2-64046`.**

<img width="735" alt="Captura de pantalla 2024-11-01 a la(s) 11 40 14 p m" src="https://github.com/user-attachments/assets/d860f2f2-0bcc-4168-a935-7198cd37737b">

