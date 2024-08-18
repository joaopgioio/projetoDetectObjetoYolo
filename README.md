# Documentação Detalhada do Sistema de Detecção de Objetos com Feedback em Áudio

Este projeto visa construir um sistema de detecção de objetos em tempo real que não apenas identifica objetos em um fluxo de vídeo da webcam, mas também fornece feedback audível em português sobre os objetos detectados. Essa funcionalidade pode ser particularmente útil em cenários onde a atenção visual do usuário está focada na cena em si, como em aplicações de assistência para deficientes visuais ou em ambientes industriais onde o operador precisa manter as mãos livres.

## Arquitetura do Sistema

O sistema é construído utilizando uma combinação de tecnologias de visão computacional, processamento de linguagem natural e interação humano-computador. A arquitetura geral é composta pelos seguintes módulos:

- **Captura de Vídeo**: A webcam do dispositivo é utilizada como fonte de entrada de vídeo, capturando frames em tempo real.

- **Detecção de Objetos**: A biblioteca Ultralytics YOLOv8, um modelo de deep learning de última geração, é empregada para detectar e classificar objetos presentes em cada frame do vídeo.

- **Geração de Anotações**: As informações sobre os objetos detectados (como nome da classe e localização) são utilizadas para gerar anotações visuais que são sobrepostas ao frame original, fornecendo ao usuário um feedback visual direto.

- **Síntese de Fala**: A biblioteca gTTS (Google Text-to-Speech) é utilizada para converter a lista de objetos detectados em uma locução em português.

- **Reprodução de Áudio**: O áudio gerado é reproduzido através do sistema de áudio do dispositivo, fornecendo ao usuário um feedback audível sobre os objetos detectados.

## Implementação

A implementação do sistema é realizada em Python, utilizando as seguintes bibliotecas:

- **Ultralytics**: Para a detecção de objetos.
- **PIL (Python Imaging Library)**: Para manipulação de imagens.
- **NumPy**: Para processamento numérico.
- **gTTS**: Para síntese de fala.
- **IPython**: Para interação com o ambiente de execução.
- **Playsound**: Para reprodução de áudio.
- **Pyglet**: Para gerenciamento de janelas e eventos.
- **PyVirtualDisplay**: Para criação de displays virtuais (útil em ambientes sem interface gráfica).

```python
! pip install --upgrade --quiet ultralytics
!pip install gTTS
#!pip install googletrans==4.0.0-rc1
!pip install --upgrade googletrans
!pip install deep-translator
!pip install playsound
!pip install pygobject
!pip install pyglet
!apt-get install -y xvfb # Install Xvfb
!pip install pyvirtualdisplay # Install a Python wrapper for Xvfb
```

O código é estruturado em diversas funções, cada uma com uma responsabilidade específica:

- `start_stream()`: Inicializa o stream de vídeo da webcam utilizando JavaScript.
```JavaScript
def start_stream():
    js = Javascript(f'''
    const IMG_SHAPE = {IMG_SHAPE};
    const IMG_QUALITY = {IMG_QUALITY};
    ''' + '''
    var video;
    var div = null;
    var stream;
    var captureCanvas;
    var imgElement;
    var labelElement;

    var pendingResolve = null;
    var shutdown = false;

    function removeDom() {
        stream.getVideoTracks()[0].stop();
        video.remove();
        div.remove();
        video = null;
        div = null;
        stream = null;
        imgElement = null;
        captureCanvas = null;
        labelElement = null;
    }

    function onAnimationFrame() {
        if (!shutdown) {
            window.requestAnimationFrame(onAnimationFrame);
        }
        if (pendingResolve) {
            var result = "";
            if (!shutdown) {
                captureCanvas.getContext('2d').drawImage(video, 0, 0, IMG_SHAPE[0], IMG_SHAPE[1]);
                result = captureCanvas.toDataURL('image/jpeg', IMG_QUALITY)
            }
            var lp = pendingResolve;
            pendingResolve = null;
            lp(result);
        }
    }

    async function createDom() {
        if (div !== null) {
            return stream;
        }

        div = document.createElement('div');
        div.style.border = '2px solid black';
        div.style.padding = '3px';
        div.style.width = '100%';
        div.style.maxWidth = '600px';
        document.body.appendChild(div);

        const modelOut = document.createElement('div');
        modelOut.innerHTML = "<span>Status: </span>";
        labelElement = document.createElement('span');
        labelElement.innerText = 'No data';
        labelElement.style.fontWeight = 'bold';
        modelOut.appendChild(labelElement);
        div.appendChild(modelOut);

        video = document.createElement('video');
        video.style.display = 'block';
        video.width = div.clientWidth - 6;
        video.setAttribute('playsinline', '');
        video.onclick = () => { shutdown = true; };
        stream = await navigator.mediaDevices.getUserMedia(
            {video: { facingMode: "environment"}});
        div.appendChild(video);

        imgElement = document.createElement('img');
        imgElement.style.position = 'absolute';
        imgElement.style.zIndex = 1;
        imgElement.onclick = () => { shutdown = true; };
        div.appendChild(imgElement);

        const instruction = document.createElement('div');
        instruction.innerHTML =
            '<span style="color: red; font-weight: bold;">' +
            'When finished, click here or on the video to stop this demo</span>';
        div.appendChild(instruction);
        instruction.onclick = () => { shutdown = true; };

        video.srcObject = stream;
        await video.play();

        captureCanvas = document.createElement('canvas');
        captureCanvas.width = IMG_SHAPE[0]; //video.videoWidth;
        captureCanvas.height = IMG_SHAPE[1]; //video.videoHeight;
        window.requestAnimationFrame(onAnimationFrame);

        return stream;
    }
    async function takePhoto(label, imgData) {
        if (shutdown) {
            removeDom();
            shutdown = false;
            return '';
        }

        var preCreate = Date.now();
        stream = await createDom();

        var preShow = Date.now();
        if (label != "") {
            labelElement.innerHTML = label;
        }

        if (imgData != "") {
            var videoRect = video.getClientRects()[0];
            imgElement.style.top = videoRect.top + "px";
            imgElement.style.left = videoRect.left + "px";
            imgElement.style.width = videoRect.width + "px";
            imgElement.style.height = videoRect.height + "px";
            imgElement.src = imgData;
        }

        var preCapture = Date.now();
        var result = await new Promise((resolve, reject) => pendingResolve = resolve);
        shutdown = false;

        return {
            'create': preShow - preCreate,
            'show': preCapture - preShow,
            'capture': Date.now() - preCapture,
            'img': result,
        };
    }
    ''')
    display(js)

def take_photo(label, img_data):
    data = eval_js(f'takePhoto("{label}", "{img_data}")')
    return data
```

- `js_response_to_image()`: Converte a resposta do JavaScript em um objeto de imagem PIL.
```JavaScript
def js_response_to_image(js_response) -> Image.Image:
    _, b64_str = js_response['img'].split(',')
    jpeg_bytes = b64decode(b64_str)
    image = Image.open(io.BytesIO(jpeg_bytes))
    return image
```

- `turn_non_black_pixels_visible()`: Ajusta a transparência das anotações.
```JavaScript
def turn_non_black_pixels_visible(rgba_compatible_array: np.ndarray) -> np.ndarray:
    rgba_compatible_array[:, :, 3] = (rgba_compatible_array.max(axis=2) > 0).astype(int) * 255
    return rgba_compatible_array
```

- `black_transparent_rgba_canvas()`: Cria um canvas transparente para as anotações.
```JavaScript
def black_transparent_rgba_canvas(w, h) -> np.ndarray:
    return np.zeros([w, h, 4], dtype=np.uint8)
```

- `draw_annotations_on_transparent_bg()`: Desenha as anotações sobre o canvas transparente.
```JavaScript
def draw_annotations_on_transparent_bg(detection_result: Results) -> Image.Image:
    black_rgba_canvas = black_transparent_rgba_canvas(*detection_result.orig_shape)
    transparent_canvas_with_boxes_invisible = detection_result.plot(font='verdana', masks=False, img=black_rgba_canvas)
    transparent_canvas_with_boxes_visible = turn_non_black_pixels_visible(transparent_canvas_with_boxes_invisible)
    image = Image.fromarray(transparent_canvas_with_boxes_visible, 'RGBA')
    return image
```

O loop principal do sistema captura frames da webcam, processa cada frame com o modelo YOLOv8, gera as anotações, sintetiza a fala com a lista de objetos detectados e reproduz o áudio resultante.

- `img_data = ''`: Inicializa uma variável chamada img_data como uma string vazia. Esta variável será usada para armazenar dados de imagem codificados em Base64, que serão enviados de volta para o navegador para exibir a imagem com as anotações.

- `while True:`: Inicia um loop infinito. O sistema continuará capturando frames da webcam e processando-os até que o usuário interrompa o processo (por exemplo, fechando a janela do navegador).

- `js_response = take_photo('Capturando...', img_data)`: Chama a função take_photo (definida em outro lugar do código) para capturar um frame da webcam. O primeiro argumento, 'Capturando...', é uma label que pode ser exibida no navegador durante a captura. O segundo argumento, img_data, contém os dados da imagem anterior (com anotações) para serem exibidos no navegador até que o próximo frame seja processado.

- `if not js_response:`: Verifica se a função take_photo retornou um valor vazio. Isso pode acontecer se o usuário interromper o stream de vídeo no navegador.

- `break`: Se js_response for vazio, o loop é interrompido, encerrando o processo de captura e processamento.

- `captured_img = js_response_to_image(js_response)`: Se js_response não for vazio, ele contém os dados do frame capturado. A função js_response_to_image é chamada para converter esses dados em um objeto de imagem PIL (Python Imaging Library), que pode ser usado para processamento posterior.

Em resumo, este trecho de código:
```python
start_stream()
img_data = ''
while True:
    js_response = take_photo('Capiturando...', img_data)
    if not js_response:
        break
    captured_img = js_response_to_image(js_response)

    def traduzir_lista(lista_ingles):
      """
      Traduz uma lista de palavras do inglês para o português.

      Args:
        lista_ingles: Uma lista de palavras em inglês.

      Returns:
        Uma lista de palavras em português.
      """

      tradutor = Translator()
      lista_portugues = []
      for palavra in lista_ingles:
        #traducao = tradutor.translate(palavra, dest='pt')
        traducao = GoogleTranslator(source='en', target='pt').translate(palavra)
        lista_portugues.append(traducao)
      return lista_portugues


    for detection_result in PRE_TRAINED_MODEL(source=np.array(captured_img), verbose=False):
        annotations_img = draw_annotations_on_transparent_bg(detection_result)

        results = PRE_TRAINED_MODEL(source=np.array(captured_img), verbose=False)

        # Extract detected object names
        objetos_detectados = [result.names[int(c)] for result in results for c in result.boxes.cls]

        # Crie uma frase com os objetos detectados

        if isinstance(objetos_detectados, list):
          objetos_detectados_portugues = traduzir_lista(objetos_detectados)
          #print(objetos_detectados_portugues)  # Output: ['gato', 'cão', 'pessoa']
        else:
          frase_objetos = "No objects detected" # Provide a message when no objects are found

        #print(objetos_detectados)
        #print(objetos_detectados_portugues)

        # Crie uma frase com os objetos detectados
        frase_objetos = "" + ", ".join(objetos_detectados)
        frase_objetos_portugues = "" + ", ".join(objetos_detectados_portugues)


        # Crie o áudio usando gTTS
        tts = gTTS(text=frase_objetos, lang='pt')
        tts_portugues = gTTS(text=frase_objetos_portugues, lang='pt')

        # Salve o áudio em um arquivo
        tts.save("objetos_detectados.mp3")
        tts_portugues.save("objetos_detectados_portugues.mp3")

        def falar_objetos():
            # Use the display module from IPython to play the audio
            display(Audio("objetos_detectados.mp3", autoplay=True))

            # Aguarda 2 segundos antes de continuar
            time.sleep(len(objetos_detectados)+2)
            # Use the display module from IPython to play the audio
            display(Audio("objetos_detectados_portugues.mp3", autoplay=True))

            # Aguarda 2 segundos antes de continuar
            time.sleep(5)

        # Salve o áudio em um arquivo temporário
        #with open('temp_audio.mp3', 'wb') as f:
        #    tts.write_to_fp(f)

        # Reproduza o áudio do arquivo temporário
        #playsound('temp_audio.mp3')



        # Use pyglet to play the audio file
        #music = pyglet.media.load('objetos_detectados.mp3', streaming=False)
        #music.play()
        #pyglet.app.run()

        # Close the window after audio playback
        #window.close()




        with io.BytesIO() as buffer:
            annotations_img.save(buffer, format='png')
            img_as_base64_str = str(b64encode(buffer.getvalue()), 'utf-8')
            img_data = f'data:image/png;base64,{img_as_base64_str}'
    falar_objetos()
 ```   

Inicia um loop que captura frames da webcam continuamente.
Envia os dados da imagem capturada para processamento.
Interrompe o loop se o usuário interromper o stream de vídeo.
Converte os dados do frame em um objeto de imagem PIL para uso posterior no processo de detecção de objetos e geração de anotações.


## Utilização

Para utilizar o sistema, basta executar o código em um ambiente Python com as bibliotecas necessárias instaladas. O sistema iniciará a captura de vídeo da webcam e fornecerá feedback visual e audível sobre os objetos detectados em tempo real.

![image](https://github.com/user-attachments/assets/c707667e-a062-450f-b69c-b735675c442e)



## Considerações Finais

Este sistema demonstra o potencial da combinação de visão computacional e processamento de linguagem natural para criar aplicações interativas e acessíveis. O feedback audível em tempo real sobre os objetos detectados pode ser aplicado em diversos cenários, como:

- **Auxílio a deficientes visuais**: Fornecendo informações sobre o ambiente ao redor.
- **Aplicações industriais**: Permitindo que operadores mantenham as mãos livres enquanto recebem informações sobre objetos em seu campo de visão.
- **Sistemas de segurança**: Alertando sobre a presença de objetos ou pessoas em áreas restritas.

O código pode ser facilmente adaptado e expandido para atender a requisitos específicos de diferentes aplicações.
