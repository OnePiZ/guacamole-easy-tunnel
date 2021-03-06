# Guacamole Easy Tunnel

Guacamole Easy Tunnel is a simple way to integrate remote display (RDP, VNC, Hyper-V VMConnect) or console (K8S, SSH, Telnet) functionality into your application. 

Solution is based on the awesome [Guacamole](http://guacamole.apache.org/) library and stripes all UI related stuff from the [guacamole-client](https://github.com/apache/guacamole-client), leaving only websocket and plain HTTP connectivity with connection parameters taken from the AES encrypted JSON dictionary passed via the payload parameter in a query string. 

Below you can find examples of encrypting the payload string and basic template for the frontend client.

## Usage

### Docker Image

The image is hosted on a [Docker Hub](https://hub.docker.com/r/mindcollapse/guacamole-easy-tunnel).

The demo server can be started by running `docker-compose up`.

### Payload Encryption Example (C#)
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using Newtonsoft.Json;

namespace Guacamole.Helpers
{
    /// <summary>
    /// Helper to encrypt payload dictionary to string.
    /// </summary>
    public static class PayloadEncryptor
    {
        private static readonly MD5CryptoServiceProvider Md5 = new MD5CryptoServiceProvider();
        
        private static byte[] GetIv(string iv) =>
            Md5.ComputeHash(Encoding.ASCII.GetBytes(iv))
                .Take(16)
                .ToArray();

        private static byte[] GetKey(string key) => 
            Md5.ComputeHash(Encoding.ASCII.GetBytes(key));

        /// <summary>
        /// Encrypt payload for RDP GW.
        /// </summary>
        /// <param name="key">Encryption key.</param>
        /// <param name="payload">Encryption IV.</param>
        /// <param name="payload">Key values of connection parameters.</param>
        public static string EncryptPayload(string key, string iv, Dictionary<string, string> payload)
        {
            var cipher = new RijndaelManaged();

            var encryptor = cipher.CreateEncryptor(
                GetKey(key),
                GetIv(iv));

            var payloadString = JsonConvert.SerializeObject(payload);
            var payloadBuffer = Encoding.UTF8.GetBytes(payloadString);

            return Convert.ToBase64String(encryptor.TransformFinalBlock(
                payloadBuffer, 0, payloadBuffer.Length));
        }
    }
}
```

The list of the per-protocol parameters can be found in the [official documentation](https://guacamole.apache.org/doc/gug/configuring-guacamole.html#connection-configuration).

The only required parameters are:

* protocol
* hostname
* port

##  Frontend Example 

```html
<html>
    <head>
        <script src="https://cdn.jsdelivr.net/npm/guacamole-common-js@1.2.0/dist/guacamole-common.min.js"></script>
    </head>

    <body>
        <div id="display" style="width: 1024px; height: 768px; display: inline-block; background-color: black;"></div>
    </body>

    <script>
         const tunnel = new Guacamole.ChainedTunnel(
            new Guacamole.HTTPTunnel("http://localhost:8080/tunnel"),
            new Guacamole.WebSocketTunnel("http://localhost:8080/websocket-tunnel")
        );

        const client = new Guacamole.Client(tunnel);
    
        client.onerror = (error) => {
            console.error(error);
        };

        window.onunload = () => {
            client.disconnect();
        };

        document.getElementById("display").append(
            client.getDisplay().getElement());

        try {
            client.connect("?payload=" + "base64 encoded AES encrypted JSON dictionary with connection parameters generated by backend");
        } catch (error) {
            console.error(error);
        }

        let mouse = new Guacamole.Mouse(client.getDisplay().getElement());

        mouse.onmousedown = mouse.onmouseup = mouse.onmousemove = (mouseState) => {
            client.sendMouseState(mouseState);
        };

        let keyboard = new Guacamole.Keyboard(document);

        keyboard.onkeydown = (keysym) => {
            client.sendKeyEvent(1, keysym);
            return false;
        };

        keyboard.onkeyup = (keysym) => {
            client.sendKeyEvent(0, keysym);
            return false;
        };
    </script>
</html>
```

Please pay attention to the `client.connect("?payload=" + "base64 encoded AES encrypted JSON dictionary with connection parameters generated by backend");`.

 