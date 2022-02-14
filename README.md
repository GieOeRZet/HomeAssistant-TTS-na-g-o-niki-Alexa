Nie jest to mój pomysł lecz zlepek informacji znalezioncyh w sieci, między innymi:
https://mic22.medium.com/make-alexa-speak-any-language-with-a-little-help-of-home-assistant-acc71d98679e
Niestety powyższy sposób już nie działa gdyż Amazon zezwala jedynie na odtwarzanie komunikatów/plików poprzez NOTIFY a nie jak wcześniej przez media-player

Aby TTS działał, konieczne jest dostęp do HA spoza sieci lokalnej (w moim przypadku jest to NabuCasa)
niezbędne dodatki i integracje:
- AppDaemon (do pobrania ze sklepu z dodatkami)
- Alexa Media Player (integracja do zainstalowania przez HACS)

Na początek jeżeli nie mamy AppDaemon'a to musimy go zainstalować.
Najprościej aby to zrobić idziemy do:
Konfiguracja -> Dodatki, kopie zapasowe oraz Supervisor -> Sklep z dodatkami
W konfiguracji AppDaemon'a wpisujemy:

    
    init_commands: []
    python_packages:
      - gTTS
      - pydub
    system_packages:
      - ffmpeg

W folderze <i><b>config/www</b></i> tworzymy folder <i><b>tts</b></i>

Przechodzimy do folderu <b><i>/config/appdaemon/apps</b></i> gdzie tworzymy plik o nazwie  <b><i>alexa_google_tts.py</b></i> z następującą zawartością:

	import appdaemon.plugins.hass.hassapi as hass
	from gtts import gTTS
	import os
	import hashlib
	from pydub import AudioSegment
	import shutil

	class AlexaGoogleTTS(hass.Hass):

		def initialize(self):
			self.log("Alexa Google TTS started")
			self.listen_state(self.text_input_handler, self.args["text_input"]) # listening for a change of the text input
			self.listen_state(self.tts_output_handler, self.args["tts_output"]) # triggering TTS script when audio file is ready

		def text_input_handler(self, entity, attribute, old, new, kwargs):
			value = new

			if value != "":
				self.log(value)

				filename = hashlib.md5(value.encode('utf-8')).hexdigest() #  md5 hash generated form the input
				filepath = '/config/www/tts/' + filename + '.mp3'
				tmp_filepath = '/config/www/tts/' + filename + '_tmp.mp3'
				out_filepath = '/config/www/tts/out.mp3'

				if not os.path.isfile(filepath): # if such file doesn't exist request TTS service and prepare audio
					tts = gTTS(value, lang='pl', slow=False) # change lang='pl' to desired language
					tts.save(tmp_filepath)

					sound = AudioSegment.from_file(tmp_filepath)
					os.remove(tmp_filepath)
					sound += 8 # increase volume by 8dB
					sound = sound.speedup(1.3) # speedup +30%
					sound = sound.set_frame_rate(16000) # Echo can handle 24k but some older device (even older Dot gen 3) are able to play just 16k
					sound.export(filepath, format="mp3", bitrate="48k") # save as mp3 with 48k bitrate

				shutil.copy(filepath, out_filepath)

				self.set_state(self.args["tts_output"], state = filename) # when all is fine just update tts_output to trigger method below

		def tts_output_handler(self, entity, attribute, old, new, kwargs):
			value = new

			if value != "":
				self.turn_on("script.google_tts") # run the player script (standard HA script, you'll add it later on)

				self.set_state(self.args["tts_output"], state = "") # clear the fields
				self.set_state(self.args["text_input"], state = "")

Następnie edytujemy <b><i>/config/appdaemon/apps/apps.yaml</b></i> aby wyglądał jak poniżej:

	---
	alexa_google_tts:
		module: alexa_google_tts
		class: AlexaGoogleTTS
		text_input: input_text.alexa_google_tts_text_input
		tts_output: input_text.alexa_google_tts_output

Kolejnym krokiem jest stworzenie w HA pomocników. Dodajemy w pliku configuration.yaml nastepujący kod:

	input_text:
		alexa_google_tts_text_input:
			name: Alexa Google TTS text input
			max: 255
			initial: ""
		alexa_google_tts_output:
			name: Alexa Google TTS output
			max: 255
			initial: ""

	input_select:
		alexa_device_for_tts:
			name: Echo device for TTS
			options: # poniżej podajemy encje głośników stworzone przez integrację Alexa Media Player
				- media_player.living_room_echo
				- media_player.bedroom_echo
				- media_player.bathroom_echo
			initial:
				media_player.living_room_echo
				
Edytujemy plik <b><i>scripts.yaml</b></i>
i dodajemy następujący skrypt:

	google_tts:
		alias: Google TTS
		mode: single
		sequence:
		- service: notify.alexa_media
			data:
				target: media_player.living_room_echo
				message: <audio src='https://twoj_adres_dostępu_do_HA_spoza_sieci_lokalnej.ui.nabu.casa/local/tts/out.mp3'/>
				data:
					type: tts

oraz tworzymy kartę w dashboardzie:

	entities:
		- entity: input_select.alexa_device_for_tts
		- entity: input_text.alexa_google_tts_text_input
		- entity: input_text.alexa_google_tts_output
	show_header_toggle: false
	title: TTS
	type: entities

