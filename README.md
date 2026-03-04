# microfrontends-module-federation
I don't like frontend at all, but maybe can be of help checking this topic.


En este repositorio veremos el tópico de los microfrontends. Si es algo
que debo de empezar a checar.

<img width="972" height="624" alt="image" src="https://github.com/user-attachments/assets/fe16a415-6905-46e3-891e-47c7602214a1" />


Los microfrontends es el mismo enfoque de microservicios... pero en el front. Así de simple. La herramienta que manipularemos en esta práctica
será webpack-module-federation, hay otra que es el single spa pero a mi me piden el otro.

La complejidad de los microfrontends recae en: como comunicar eventos o comportamientos entre componentes, digamos en vue es ref, 
en angular son signals, entonces tienes que estarle jugando al ping pong para eso, y probablemente en mi imaginación, las pruebas unitarias
si han de ser un infierno.

Acompañenme en esta aventura. Será de los últimos tópicos en estudiar antes de empezar a trabajar.


Haremos por el momento una práctica sencillita, porque hay otro tema que me urge ver.

Primero lo primero. De module federation hay dos conceptos: consumers y providers.

Los providers son los microfrontends que estaremos desarrollando. Los consumers el intermediario que los fusiona y los expone.

En microfrontends se menciona mucho el concepto de shell. Los consumers son nuestro shell en este caso. Lo más dificultoso al principio es 
la configuración, porque he de admitir que la documentación de estos componentes es UN ASCO. Honestamente, antes de la IA tenias que ver tutoriales
y rezar que te saliera algo. Cada dia se te fuerza más a usar IA. Sería increible que los mantainers crearan buena documentación.

Volviendo al tema, escribiré los puntos que más me costaron configurar:

Cuando crees el setup con npm create module-federation@latest, te aparecerá que si lo quieres
en modern.js o rsbuild, hasta ahora y porque apenas aprendo el marco, yo uso el modern porque ni sabia que estaba escogiendo.

Seleccionas primero el provider, genera el cascarón de la aplicación. Por default lo genera en React, porque me parece module federation
apunta más por el mismo framework.

<img width="1186" height="775" alt="image" src="https://github.com/user-attachments/assets/917b9b7f-5884-4c73-acf2-74255339b95d" />

Te ahorraré unos pasos, copia y pega esto en el module-federation.config.ts

````main.ts
import { createModuleFederationConfig } from '@module-federation/modern-js-v3';

export default createModuleFederationConfig({
  name: 'mf_microfrontend_practice',
  filename: 'remoteEntry.js',
  exposes: {
    './ProviderCustom': './src/components/ProviderComponent.tsx',
  },
  shared: {
    react: { singleton: true },
    'react-dom': { singleton: true },
  },
  manifest:true,
  dts: {
    generateTypes: true,
  },
});

````

El manifest true es para que se te genere el archivo mf-manifest.json que servirá para exponer tu micro al shell. El
dfs.generatedTypes es para exponer los tipos de componentes que creas desde tu micro, y conectarlos para que tu consumidor los pueda instanciar.

Después haz las modificaciones pertinentes en digamos... el componente de ejemplo y en modern.config.ts pega esto:

````main.ts
import { appTools, defineConfig } from '@modern-js/app-tools';
import { moduleFederationPlugin } from '@module-federation/modern-js-v3';

// https://modernjs.dev/en/configure/app/usage
export default defineConfig({
  plugins: [appTools(), moduleFederationPlugin()],
  server: {
    port: 3001,
  },
  tools: {
    // Modern.js usa esta sección para extender la config del servidor de desarrollo
    devServer: {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, PATCH, OPTIONS',
        'Access-Control-Allow-Headers': '*',
      },
    },
  },

});


````

La configuracion del tools dev sirve para que puedas exponer tu micro correctamente, y el consumer detecte
el archivo mf-manifest.js

Ahora, un paso escencial para poder conectar tus micros es ejecutar

````main.sh
npm run build
````

Esto te generará los archivos remoteEntry.js y el manifest que servirán para exponer y vincular tu micro. La manera
más fácil de saber si tu micro está expuesto es por dos checks:

+Check de manifest*

<img width="2558" height="1439" alt="image" src="https://github.com/user-attachments/assets/8a3f1deb-1376-4249-8a26-de37856047b8" />


*Check del componente unitario*

<img width="2545" height="1439" alt="image" src="https://github.com/user-attachments/assets/3418c0b5-f2c9-4ceb-9c3e-b2e826a28725" />

Si no lo haces vas a batallar con este error

<img width="1186" height="552" alt="image" src="https://github.com/user-attachments/assets/137d2937-3e30-4c73-a5b5-a304118307df" />


Para generar el consumer (shell) es lo mismo desde el npm create escoges consumer en vez de provider, te dará el siguiente cascarón, 
y ejecutas de nuevo el npm run build.

<img width="1354" height="1013" alt="image" src="https://github.com/user-attachments/assets/27708b0a-1394-488c-8a5b-7a1233e31abb" />

En el archivo module-federation.config.ts del consumer declaras lo siguiente:

````main.ts
import { createModuleFederationConfig } from '@module-federation/modern-js-v3';

export default createModuleFederationConfig({
  name: 'mf_microfrontend_consumer',
  remotes: {
    'provider': 'rslib_provider@https://unpkg.com/module-federation-rslib-provider@latest/dist/mf/mf-manifest.json',
    'remote': 'mf_microfrontend_practice@http://localhost:3001/mf-manifest.json'
  },
  shared: {
    react: { singleton: true },
    'react-dom': { singleton: true },
  },
  dts: {
    consumeTypes: true,
  },
});
````

Declaras un nuevo scope remote e introduces el endpoint de tu micro front para conectar tu micro unitario al
shell.




Y para adaptarlo, usa el siguiente snippet:

````main.ts
import { Helmet } from '@modern-js/runtime/head';
import './index.css';
import { lazy } from 'react';
//import Provider from 'provider';
import { loadRemote } from '@module-federation/enhanced/runtime';


const IndexCustom = lazy(() =>
  loadRemote('mf_microfrontend_practice/IndexCustom').then((mod: any) => ({
    default: mod.default
  }))
);

const Index = () => (
  <div className="container-box">
    <Helmet>
      <link
        rel="icon"
        type="image/x-icon"
        href="https://lf3-static.bytednsdoc.com/obj/eden-cn/uhbfnupenuhf/favicon.ico"
      />
    </Helmet>

    <div className="landing-page-2">
      dasdasd
    </div>

    <div className='remoteDom'>
      <IndexCustom/>
    </div>
  </div>
);

export default Index;

````

Lo adaptas como un componente normal, y lo invocas en tag que quieras, y recuerda que al adaptar el componente a tu consumer, 
también pueden chocar las clases padre por el css.

*Check del componente integrado en consumer (patrón microfrontend)*
<img width="2447" height="1439" alt="image" src="https://github.com/user-attachments/assets/2cd7a580-e854-4700-8dac-671dd17e48f4" />

Por cada update que hagas en tu componente, se refleja en automático en tu componente, pero si vas a crear otro componente, tendrás que declararlo
en tus configs siempre, uno por uno y ejecutar npm run build para sincronizar los tipados:

<img width="855" height="220" alt="image" src="https://github.com/user-attachments/assets/eb24ec4e-eaa2-4366-bcf8-7a6853f2ac86" />

Esto es lo básico de un microfrontend. ¿Ahora que pasa si quiero adaptar el componente de otro framework?

El proyecto más próximo que tengo es un micro de angular. Vamos a ver que tal, en base a un desglose, veremos como conectar eventos.

## Avances 03/03/2026 

Al parecer modern.js está orientado más al desglose de un monólito react que sobre una comunicación entre distintos frameworks.
Hasta ahora llevo un dolor de cabeza grande entre claude, la virgen y lo que sea. En primeras está cañón y creo no hay compatibilidad entre 
@angular-architects/module-federation y module-federation v2.0. Entonces, el approach más antiguo entre micros es por iframe:

Otra cuestión es que rsbuild actualmente tiene una apertura de más facilidad a adaptar los multi frameworks, modern.js es más orientado a
descomponer en pequeños fragmentos una app en react.

El approach más básico es el iframe

<img width="2413" height="1439" alt="image" src="https://github.com/user-attachments/assets/2c3e7f2a-4155-49d8-a917-e455afd1300d" />

Funciona normal pues... pero hasta ahora no es lo que yo deseo.

El module federation de rs build del module federation no tiene compatibilidad hasta ahora con react, tendria que ser por webpack puro por lo 
que estoy checando. Por lo que veo el framework que tiene esa finalidad es el single spa.

Después de promptear por un buen rato, me di cuenta de algo:

No hay adaptación natural entre el module federation de angular y react. No pierdan su tiempo. Tal parece que los approaches 
si deben ser por wireframes para la orquestación de algunos tiempos:

<img width="2267" height="1355" alt="image" src="https://github.com/user-attachments/assets/264a941d-2391-414c-9f9f-5d6295822f17" />

Y la comunicación involucra usar el postmessage 

<img width="2559" height="1439" alt="image" src="https://github.com/user-attachments/assets/e1380616-65ab-4b6b-b965-e7aa59e8af0a" />

No es lo que esperaba, si es un campo nuevecito para mi este rollo.

Necesitaré más tiempo para ir acoplando este pex, pero al menos ya vi cual fue el tema masomenos...

````masn.md
| Remote | Host | ¿Funciona? | Condición |
|---|---|---|---|
| React (Webpack) | React (Webpack) | ✅ | Misma versión MF |
| React (Rsbuild MF v2) | React (Rsbuild MF v2) | ✅ | Mismo runtime |
| Vue (Webpack) | React (Webpack) | ✅ | Mismo bundler |
| Angular (Webpack MF v1) | React (Webpack MF v1) | ✅ | Mismo bundler y versión |
| Angular (Webpack MF v1) | React (Rsbuild MF v2) | ❌ | Runtime mismatch — fue tu problema |
| Angular (Rsbuild/Nx) | React (Rsbuild MF v2) | ✅ | Mismo runtime |
| Cualquier framework | Cualquier framework | ✅ | Siempre que compartan el mismo runtime de MF |
````

Hasta ahora lo que hemos logrado conectar son componentes remotos de react, para el tema de no acoplar todo sobre un solo code base
y no tengas merge conflicts, y un iframe para conectar un componente externo de angular

Debo checar más sobre webpack. Pero creo que si hay muchos temas de compatibilidad entre estos approaches que intento hacer con module
federation ,native federation, creo solo sirven para desglosar un monolito y separar, pero no para integrar multi frameworks, eso solo
con iframes o single spa....

Una manera dura de aprenderlo, hasta claude anda fastidiado.

<img width="2540" height="1439" alt="image" src="https://github.com/user-attachments/assets/e9f54b60-c322-4dcb-917f-a54a94863ae7" />

Porque quise hacer un wrapper pero no, asi no jala aun asi

<img width="1523" height="1030" alt="image" src="https://github.com/user-attachments/assets/403cced5-cc0b-4ea3-911c-c7be2c63d5ec" />

Y desde angular también he intentado con los webcomponents pero aunque se diga que si, no hay compatibilidad realmente.

## Avances 04/03/2026

Claude me dio una alternativa interesante que es la herramienta NX, con la cual algunos colaboradores cercanos de module federation
trabajan en conjunto. Es una herramienta que sirve para establecer un ambiente de desarrollo monorepo, orientado a la conexión de microfrontends
por medio de un solo runtime de module federation y la abstracción de dependencias comunes entre los dos o más proyectos separados.

Es una manera efectiva de conectar un host con react a angular o vue o viceversa, en cualquier combinación posible.

Te dejo los comandos a ejecutar:

````main.sh
npx create-nx-workspace@latest mi-workspace --preset=empty

cd mi-workspace

# Agregar Angular
$env:NX_IGNORE_UNSUPPORTED_TS_SETUP="true"
npx nx add @nx/angular

# Agregar React  
npx nx add @nx/react

# React como host
npx nx g @nx/react:host mfreacthost --remotes=mfangularremote --bundler=rspack

# Angular como remote
npx nx g @nx/angular:remote mfangularremote --host=mfreacthost --bundler=rspack

## ejecuta el host y levantará react y angular
npx nx serve mfreacthost
````

En consola te debe aparecer así, significa que ya arranco el proyecto:

<img width="1656" height="500" alt="image" src="https://github.com/user-attachments/assets/93582e4f-40a8-48b7-94f9-b0c87203886b" />


Y ya te carga ambos componentes, es un parent app de react conectado a un angular app como componente:

<img width="2401" height="910" alt="image" src="https://github.com/user-attachments/assets/0fb25068-4856-46ef-8330-1918b2197ae2" />

<img width="2076" height="547" alt="image" src="https://github.com/user-attachments/assets/6cad0633-c99e-47ba-b5ed-3707f02cf467" />


Los puedes conectar por medio de librerias

````main.ts
// libs/event-bus/src/index.ts
import { Subject } from 'rxjs';

export const formSubmit$ = new Subject<any>();

import { formSubmit$ } from 'event-bus';
onSubmit() {
  formSubmit$.next(this.userModel.value);
}

/*Y en react*/
import { formSubmit$ } from 'event-bus';
useEffect(() => {
  const sub = formSubmit$.subscribe(data => console.log(data));
  return () => sub.unsubscribe();
}, []);

//También por medio de custom events:
// En el componente Angular
onSubmit() {
  window.dispatchEvent(new CustomEvent('angular:form-submit', {
    detail: { data: this.userModel.value },
    bubbles: true
  }));
}

// En React host
useEffect(() => {
  const handler = (e: any) => {
    console.log('Datos de Angular:', e.detail.data);
  };
  window.addEventListener('angular:form-submit', handler);
  return () => window.removeEventListener('angular:form-submit', handler);
}, []);

````

Entonces llegamos con esta introducción a las siguientes conclusiones:

-No pierdas el tiempo queriendo adaptar MF's de diversos aplicativos sin un bridge de los pocos que ofrecen

-Single spa y Nx te solucionan el tema de los cross framework

-Angular acrhitect y modern js solo aplica en modalidades del desglose de de tu monolito. Aunque por lo que vemos
pudieramos combinar el MF con el mismo NX y lograr un desglose en los monorepos también, pero si viene siento más complejidad aún

-Iframes puede ser una solución más rápida y fácil para adaptar cross, así como el proteger ese componente expuesto.

Creo que hemos logrado dominar lo básico de microfrontends, ya cuando chambeé ya estudiaré codigos ajenos.

