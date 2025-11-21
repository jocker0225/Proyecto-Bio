🟦 VISIÓN GENERAL

GaitPy es una librería Python (no existe en Dart).
Flutter NO puede ejecutar Python nativamente, así que necesitamos:

👉 Flutter → motor Python (Starflut) → ejecuta tu script con GaitPy → devuelve resultados a Flutter

Esto permite:

Procesar señales de acelerómetros en el propio celular

Extraer cadencia, velocidad, longitud de paso, etc.

Trabajar OFFLINE

Usar código Python sin reescribirlo en Dart

🟦 PASO 1 — Instalar el plugin Starflut en Flutter

Starflut es un “puente” que permite integrar lenguajes como Python en apps Flutter.

🔧 ¿Qué hace?

Empaca un intérprete Python dentro de la app

Permite ejecutar archivos .py

Enviar y recibir datos (strings, JSON, listas)

📌 Cómo instalarlo

Abre tu archivo pubspec.yaml y agrega:

dependencies:
  flutter:
    sdk: flutter
  starflut: ^2.0.0


Luego ejecuta:

flutter pub get

🟦 PASO 2 — Crear estructura para Python dentro del proyecto Flutter

En tu carpeta del proyecto, crea:

/assets/python/
        gait_engine.py
        requirements.txt

📌 ¿Qué significan estos archivos?
✔ gait_engine.py

Aquí colocas el código Python que procesará la marcha usando GaitPy.

✔ requirements.txt

Aquí deben listarse las dependencias Python que necesita tu script:

gaitpy
numpy
pandas
scipy
matplotlib


⚠️ IMPORTANTE:
Starflut incluye un Python minimalista, por lo que a veces tendrás que empacar las dependencias dentro del APK.

(Te explico esto más adelante en la parte de notas técnicas.)

🟦 PASO 3 — Registrar los assets Python en pubspec.yaml

Agrega:

flutter:
  assets:
    - assets/python/

❗ ¿Por qué?

Flutter necesita saber qué archivos incluir cuando genera el APK.

🟦 PASO 4 — Crear el script de análisis en Python (gait_engine.py)

Este archivo será ejecutado por la app Flutter.
Es el equivalente al script que ya usas en tu computadora, pero adaptado para ser llamado desde Flutter.

Crea assets/python/gait_engine.py con:

from gaitpy.gait import Gaitpy
import json

def run_gait(raw_path, sample_rate, height):
    gaitpy = Gaitpy(
        raw_path,
        int(sample_rate),
        v_acc_col_name='y',
        ts_col_name='timestamps',
        v_acc_units='m/s^2',
        ts_units='ms',
        flip=False
    )

    bouts = gaitpy.classify_bouts()
    feats = gaitpy.extract_features(
        float(height),
        subject_height_units='centimeter',
        classified_gait=bouts
    )

    # Convertir DataFrame -> diccionario -> JSON
    return json.dumps(feats.to_dict())

✔ ¿Qué hace este script?

Recibe:

ruta del archivo CSV

sample rate

estatura del sujeto

Usa GaitPy para:

Detectar periodos de marcha

Extraer características: velocidad, cadencia, longitud de paso, etc.

Envía los resultados a Flutter como un JSON.

🟦 PASO 5 — Crear pantalla Flutter para ejecutar el análisis

Archivo: lib/gait_screen.dart

import 'package:flutter/material.dart';
import 'package:starflut/starflut.dart';

class GaitScreen extends StatefulWidget {
  @override
  _GaitScreenState createState() => _GaitScreenState();
}

class _GaitScreenState extends State<GaitScreen> {

  String output = "Sin análisis";

  Future<void> runGait() async {
    // 1. Iniciar el motor StarCore
    StarCoreFactory starcore = await Starflut.getFactory();
    StarServiceClass service = await starcore.initSimple("test", "123", 0, 0, []);

    // 2. Cargar módulo Python que está en assets/python/gait_engine.py
    await Starflut.loadPyModule(service, "assets/python/", "gait_engine");

    // 3. Obtener referencia al intérprete Python
    StarObjectClass python = await service.importRawContext("python", "", false, "");

    // 4. Llamar a la función run_gait del archivo gait_engine.py
    var result = await python.call("run_gait", [
      "/storage/emulated/0/Download/raw.csv",   // ruta del CSV
      "128",                                    // freq muestreo
      "170"                                     // altura en cm
    ]);

    // 5. Mostrar resultados
    setState(() {
      output = result.toString();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("Evaluación de Marcha")),
      body: Padding(
        padding: EdgeInsets.all(20),
        child: Column(
          children: [
            ElevatedButton(
              onPressed: runGait,
              child: Text("Procesar Marcha"),
            ),
            SizedBox(height: 20),
            Text(output)
          ],
        ),
      ),
    );
  }
}

🟦 PASO 6 — Explicación detallada del flujo interno
1️⃣ Flutter inicializa el motor StarCore

Esto inicia un entorno que puede correr Python.

StarCoreFactory starcore = await Starflut.getFactory();

2️⃣ Se cargan los scripts Python desde /assets/python/
await Starflut.loadPyModule(service, "assets/python/", "gait_engine");


Esto hace que Python “vea” tu archivo gait_engine.py.

3️⃣ Flutter obtiene acceso al intérprete Python
StarObjectClass python = await service.importRawContext("python", "", false, "");


Esto produce un objeto que permite llamar funciones Python desde Dart.

4️⃣ Flutter ejecuta:
python.call("run_gait", [...]);


Y este llama al Python:

run_gait(raw_path, sample_rate, height)

5️⃣ Python analiza los datos con GaitPy

Detecta:

pasos

ciclos de marcha

velocidad

cadencia

longitud del paso

fase de balanceo y apoyo

6️⃣ Python convierte resultados a JSON y los envía a Flutter

Flutter recibe algo así:

{
  "gait_speed": 1.12,
  "cadence": 108,
  "stride_length": 1.26
}

🟦 PASO 7 — Formato del archivo CSV

Debe tener:

timestamps (ms)	y (m/s^2)
1653143512345	9.81
1653143512353	10.02
...	...
🟦 PASO 8 — ¿Qué falta para que funcione al 100%?
✔ Incluir dependencias Python dentro del APK

Starflut necesita que empaques las librerías Python como:

numpy

pandas

gaitpy

Puedo darte instrucciones exactas para empaquetarlas (más avanzado pero necesario).

✔ Permisos de lectura de archivos en Android

Para leer el CSV:

READ_EXTERNAL_STORAGE
WRITE_EXTERNAL_STORAGE

✔ Pantalla Flutter para seleccionar el archivo

Puedo generarla si la necesitas.
