# Projetos_Arduino
// ----------- INICIALIZAÇÃO DO SENSOR ULTRASÔNICO ----------

#include "Ultrasonic.h" // BIBLIOTECA
#define pin_trigger 11
#define pin_echo 10
Ultrasonic ultrasonic(pin_trigger, pin_echo); //INICIALIZA OS PINOS DOS SENSORES

//---------------- INICIALIZAÇÃO RTC -------------------------

#include <Wire.h> //BIBLIOTECA
#include "RTClib.h" //BIBLIOTECA
RTC_DS1307 rtc; //OBJETO DO TIPO RTC_DS1307
char daysOfTheWeek[7][12] = {"Domingo", "Segunda", "Terça", "Quarta", "Quinta", "Sexta", "Sábado"};

//---------------- INICIALIZAÇÃO DO SENSOR DE TEMPERATURA --------

#include <OneWire.h>
#include <DallasTemperature.h>
#define DS18B20 2 //DEFINE O PINO DIGITAL UTILIZADO PELO SENSOR
OneWire ourWire(DS18B20); //CONFIGURA UMA INSTÂNCIA ONEWIRE PARA SE COMUNICAR COM O SENSOR
DallasTemperature sensors(&ourWire); //BIBLIOTECA DallasTemperature UTILIZA A OneWire

// -------------- INICIALIZAÇÃO DO SENSOR LDR -------------------

#define sen_ldr A1

//------------- INICIALIZA OS PIN-----------------------------

#define bomba 7
#define luminaria 6
#define refrig 5

#define led_nivel_alto 9
#define led_nivel_baixo 8
#define led_nivel_critico 12

//-------------- VARIÁVEIS -----------------------------------

int tempo_bomba_desligada = 21; // TEMPO EM SEGUNDOS DA BOMBA EM ESPERA (intervalos).
int tempo_bomba_ligada = 20;  // quanto tempoa bomba permace ligada 
int tempo_leitura_ultra = 12;
int tempo_leitura_temperatura = 13;
int tempo_leitura_lumi = 14; 
int nivel_alto = 10; // NIVEL ALTO (cm)
int nivel_baixo = 15;  // NIVEL BAIXO (cm)

// ------------- Variaveis De controle ----------------------

int distancia;
static bool flag_bomba_ligada;
static bool flag_luminaria;
float temperatura_media;
float tempo_adicional; //<------------ TESTANDO ! Não implementado 

// --------------------- REFERENCIAS DE TEMPO ---------------

DateTime now;
DateTime ref_tempo_liga;
DateTime ref_tempo_desliga;
DateTime ref_tempo_ultra;
DateTime ref_tempo_temperatura;
DateTime ref_tempo_lumi;

//======================= FUNÇÕES SETUP/LOOP =======================

void setup() {

  start();
  strt_rtc();

}
void loop() {  

  now = rtc.now();
  leitura_temperatura ();
  check_ultra_sen();
  acionamento_bomba();
  comportamento_luminaria();
}
//======================= INICIALIZAÇÂO =======================

void start(){
  
  Serial.begin(9600);
  DateTime ref_tempo_liga = rtc.now();
  DateTime ref_tempo_ultra = rtc.now();
  DateTime ref_tempo_temperatura = rtc.now();
  DateTime ref_tempo_lumi = rtc.now(); 
  
 // ------------------- DEFINIÇOES DOS PIN --------------------
 
  pinMode(bomba, OUTPUT);
  pinMode(luminaria, OUTPUT);
  pinMode(refrig, OUTPUT);
  
  pinMode(led_nivel_alto, OUTPUT);
  pinMode(led_nivel_baixo, OUTPUT);
  pinMode(led_nivel_critico, OUTPUT);
  pinMode(sen_ldr, INPUT_PULLUP);
  
  }

  // =================== INICIALIZAÇÃO RTC =======================
  
  void strt_rtc(){

  if (! rtc.begin()) { 
      Serial.println("DS1307 não encontrado"); 
      while(1); 
    }
    if (! rtc.isrunning()) { 
      Serial.println("DS1307 rodando!");
      //rtc.adjust(DateTime(F(__DATE__), F(__TIME__))); 
      //rtc.adjust(DateTime(2018, 7, 5, 15, 33, 15)); //(ANO), (MÊS), (DIA), (HORA), (MINUTOS), (SEGUNDOS)
  }
    }
// ======================= VERIFICAÇÃO DE NÍVEL ===============

  void ultrasonic_sen(){
    
    digitalWrite(pin_trigger, LOW);
    delayMicroseconds(2); 
    digitalWrite(pin_trigger, HIGH); 
    delayMicroseconds(10);  
    digitalWrite(pin_trigger, LOW); 
    distancia = (ultrasonic.Ranging(CM));
    delayMicroseconds(10);
  }

  void check_ultra_sen(){
    
    if (now.unixtime() >= (ref_tempo_ultra.unixtime() + tempo_leitura_ultra)){
      ultrasonic_sen();
      Serial.println("lendo o nível da água ");
      if (distancia <= nivel_alto){
        // led que indica nivel alto  
        digitalWrite(led_nivel_alto, HIGH);
        digitalWrite(led_nivel_baixo, LOW);
        }
       if (distancia > nivel_alto && distancia < nivel_baixo){
          // led indica nivel baixo
          digitalWrite(led_nivel_alto, LOW);
          digitalWrite(led_nivel_baixo, LOW);
        }
       if (distancia > nivel_baixo){
        // NIVEL CRITICO
        //digitalWrite(bomba, LOW); 
        flag_bomba_ligada = false; // <-------------SE O NIVEL FOR BAIXO PARA A BOMBA ------------------<<<<<<<<<<
        digitalWrite(led_nivel_alto, LOW);
        digitalWrite(led_nivel_baixo, HIGH);
     
          }
          ref_tempo_ultra = now;
        }
      } 
// ================== CALCULO DE TEMPO ENTRE ACINAMENTOS DA BOMBA =================

int tempo_para_ligar(){

  if (estado() == 0){
  int result;
  result = (tempo_bomba_desligada /4) + tempo_bomba_desligada;
  return result;
  }else {
    return tempo_bomba_desligada;
    }
  }
  
//  ==================== COMANDOS DA BOMBA ==========================
  
  void comportamento_bomba(){
    
    static bool flag_sen;
        
    if (now.unixtime() >= (ref_tempo_liga.unixtime() + tempo_para_ligar()) && flag_bomba_ligada == false ){
      
      if (flag_sen == false){
       ultrasonic_sen(); // <------- VERIFICAR O SENSOR ULTRASÔNICO ANTES DE LIGAR A BOMBA DÁ TRABALHO KKKK ----<<<<
       Serial.println("Leitura de verificação da bomba");
       flag_sen = true;
      }
      
     if (distancia <= nivel_baixo){
      
      Serial.println(" Bomba ligada !");
      flag_bomba_ligada = true; // <----- ESSE FLAG FAZ A BOMBA LIGAR ! ---<<<<<
      ref_tempo_desliga = now;
      flag_sen = false; // <---- ESSE FLAG INDICA UMA LEITURA DO SENSOR ---<<<<
        }
      }
      
    if (now.unixtime() >= (ref_tempo_desliga.unixtime() + tempo_bomba_ligada) && flag_bomba_ligada == true){
      
      Serial.println("Bomba desligada !");
      ref_tempo_liga = now;
      flag_bomba_ligada = false;
      }  
  }

  void acionamento_bomba(){ // < ------------- comando de acionamento da bomba -------<<<<
  
    comportamento_bomba();
    
     if (flag_bomba_ligada == true){
        digitalWrite(bomba, HIGH);
    }else {
         digitalWrite(bomba, LOW);
        }
    }
    
  //===================== REFRIGERAÇÃO DA ÁGUA ===================

 float sen_temperatura(){
  
  float temp;
  sensors.requestTemperatures();
  return  temp = sensors.getTempCByIndex(0);
  
  }
  
 void leitura_temperatura (){
  
  static int cont;
  float temp;
  float maxtemp;
  float mintemp;
  float result;
  
  if (now.unixtime() >= (ref_tempo_temperatura.unixtime() + tempo_leitura_temperatura)){ 

   
    temp = sen_temperatura();
    
     if (cont == 0){
      maxtemp = temp;  
      mintemp = temp;
      cont += 1;
      }
     if (temp > maxtemp && cont != 0){
      maxtemp = temp;  
      } 
     if (temp < mintemp && cont != 0){
      mintemp = temp;
      }
     if(cont != 0 ){
      float soma;
      
      // indice[cont] = temp;
      soma = (temp * 3) + (maxtemp + mintemp);
      result = soma /5 ;
     } else {
      result = temp;
      }
    temperatura_media = result;
      
  Serial.print("Temperatura da agua: "); 
  Serial.print(temperatura_media);
  Serial.println("*C");

  ref_tempo_temperatura = now;
  acionamento_refrig();
    }
  }
  void acionamento_refrig(){
    
    if (temperatura_media >= 28.5 ){
      digitalWrite(refrig,HIGH);
      }else{
        digitalWrite(refrig,LOW);
        
        }
    }
//  =============== CONTROLE DA ILUMINAÇÃO ========================

int estado(){
   if (now.hour() >= 5 && now.hour() < 18){
       // Serial.print("Dia ");
     return 0;
        }
   if(now.hour() >= 18  && now.hour() < 5){
         // Serial.print("Noite ");
      return 1;
          }
      }
      
float sensor_ldr(){
  
   float valor;
   valor = analogRead(sen_ldr);
   return valor;
  }
  
void comportamento_luminaria(){

if (now.unixtime() >= (ref_tempo_lumi.unixtime() + tempo_leitura_lumi)){ 

  Serial.print(tempo_para_ligar());
 if (sensor_ldr() <= 300 && estado() == 0){
  
   // luminosidade alta
   digitalWrite(luminaria, LOW);
   Serial.println("Luz desligada");
  } 
 if (sensor_ldr() > 300 && estado()== 0){
  Serial.println("Luz Ligada !!");
  digitalWrite(luminaria, HIGH);
   // luminosidade baixa
  } 
  if (estado()== 1){
    
     digitalWrite(luminaria, LOW);
    }
  ref_tempo_lumi = now;  
 }
}
