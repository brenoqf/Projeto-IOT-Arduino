# Projeto IOT - 5º Semestre

## Membros do Projeto
* Breno Queiroga Faustino R.A: 22124001-3
* Rafael Levi Ramos Fernandes R.A: 22124057-5

## 1. Relatório Técnico

### Descrição do Projeto
Este projeto consiste em um sistema de controle de acesso para um compartimento seguro, desenvolvido na plataforma **Arduino Uno**. O sistema gerencia a entrada de senhas via 4 botões, valida o acesso através de uma **Máquina de Estados Finita (FSM)** e fornece feedback em tempo real ao usuário por meio de componentes visuais (LCD e LEDs), sonoros (Buzzer) e mecânicos (Servo).

| Nome | Quantidade | Componente |
| :--- | :--- | :--- |
| U1 | 1 | Arduino Uno R3 |
| U2 | 1 | LCD 16 x 2 |
| R1 | 1 | 220 Ω Resistor |
| D1 | 1 | Vermelho LED |
| D2 | 1 | Amarelo LED |
| D3 | 1 | Verde LED |
| SERVO2 | 1 | Posicional Micro Servo |
| R6, R7, R8, R10, R2, R3, R4, R5 | 8 | 1 Ω Resistor |
| S5, S1, S2, S3, S4 | 5 | Botão |
| PIEZO1 | 1 | Piezo |

---

## 2. Diagrama de Conexão

| Componente | Pino do Arduino | Tipo de Sinais | Função no Projeto |
| :--- | :---: | :---: | :--- |
| **Botão de RESET** | Pino 2 | Digital (Interrupção) | Reinicia o sistema instantaneamente para o estado Trancado e zera os erros. |
| **LED Amarelo** | Pino 3 | Digital (Saída) | Indica o estado de **ALERTA / BLOQUEADO** após 3 erros de senha. |
| **LED Vermelho** | Pino 4 | Digital (Saída) | Indica o estado de **TRANCADO** (Cofre Fechado e monitorando). |
| **LED Verde** | Pino 5 | Digital (Saída) | Indica o estado de **ABERTO** (Acesso Liberado para o usuário). |
| **Buzzer Piezo** | Pino 6 | Digital (Saída Lógica) | Emite os bipes de feedback multissensorial (常规 toque, erro e alarme). |
| **Micro Servo Motor** | Pino 7 | Digital (Controle) | Atua como o mecanismo de trava física da porta do cofre. |
| **Display LCD (D7)** | Pino 8 | Digital (Saída) | Linha de dados (High) para comunicação com o display. |
| **Display LCD (D6)** | Pino 9 | Digital (Saída) | Linha de dados (High) para comunicação com o display. |
| **Display LCD (D5)** | Pino 10 | Digital (Saída) | Linha de dados (High) para comunicação com o display. |
| **Display LCD (D4)** | Pino 11 | Digital (Saída) | Linha de dados (High) para comunicação com o display. |
| **Display LCD (E)** | Pino 12 | Digital (Saída) | Sinal de ativação de escrita (*Enable*) do display. |
| **Display LCD (RS)** | Pino 13 | Digital (Saída) | Seleção de Registro (*Register Select*) do display. |
| **Botão de Senha 1** | Pino A0 | Analógico (Entrada) | Primeiro botão de entrada da combinação secreta. |
| **Botão de Senha 2** | Pino A1 | Analógico (Entrada) | Segundo botão de entrada da combinação secreta. |
| **Botão de Senha 3** | Pino A2 | Analógico (Entrada) | Terceiro botão de entrada da combinação secreta. |
| **Botão de Senha 4** | Pino A3 | Analógico (Entrada) | Quarto botão de entrada da combinação secreta. |

---

## 3. Código Fonte

```cpp
#include <LiquidCrystal.h>
#include <Servo.h>
 
LiquidCrystal lcd(13, 12, 11, 10, 9, 8);
 
//Configurações da Senha
const int SENHA_MESTRA[] = {0, 1, 2, 3};
const int pinoReset = 2;
 
//Variáveis voláteis para garantir interrupção
volatile int tentativaUsuario[4];
volatile int indiceDigito = 0;
volatile int erros = 0;
 
//Pinos dos Botões
const int botoes[] = {A0, A1, A2, A3};
 
//Pino do Servo
const int pinoServo = 7;
Servo tranca;
 
//Pino do Buzzer
const int pinoBuzzer = 6;
 
//Estados do Cofre
enum Estados { TRANCADO, ABERTO, ALERTA };
volatile Estados estadoAtual = TRANCADO;
 
void setup() {
  lcd.begin(16, 2);
 
  pinMode(pinoReset, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(pinoReset), resetarSistema, FALLING);
 
  for(int i = 0; i < 4; i++) {
    pinMode(botoes[i], INPUT);
  }
 
  pinMode(5, OUTPUT); //LED verde
  pinMode(4, OUTPUT); //LED vermelho
  pinMode(3, OUTPUT); //LED amarelo
  pinMode(pinoBuzzer, OUTPUT); //Buzzer
 
  tranca.attach(pinoServo);
  tranca.write(0);
 
  exibirMenuInicial();
}
 
//Função de vibração do Buzzer
void emitirSom(int frequencia, int duracao) {
  tone(pinoBuzzer, frequencia, duracao);
  delay(duracao);
  noTone(pinoBuzzer);
}
 
//Função de Interrupção
void resetarSistema() {
  erros = 0;
  indiceDigito = 0;
  estadoAtual = TRANCADO;
}
 
void loop() {
  if (estadoAtual == TRANCADO) {
    digitalWrite(5, LOW);  //Desliga verde
    digitalWrite(3, LOW);  //Desliga amarelo
    digitalWrite(4, HIGH); //Liga vermelho
    tranca.write(0);
	
    //Atualiza o menu se o índice de dígitos foi resetado
    static int ultimoIndice = -1;
	if (indiceDigito == 0 && ultimoIndice != 0) {
      exibirMenuInicial();
	}
    ultimoIndice = indiceDigito;
 
    verificarBotoes();
  }
}
 
void exibirMenuInicial() {
  lcd.clear();
  lcd.print("Cofre Trancado");
  lcd.setCursor(0, 1);
  lcd.print("Senha:");
}
 
void verificarBotoes() {
  for (int i = 0; i < 4; i++) {
	if (analogRead(botoes[i]) > 800) {
      emitirSom(1000, 50);
      registrarDigito(i);
      while(analogRead(botoes[i]) > 800);
	}
  }
}
 
void registrarDigito(int botaoPressionado) {
  if (indiceDigito < 4) {
    tentativaUsuario[indiceDigito] = botaoPressionado;
    lcd.print("*");
    indiceDigito++;
 
	if (indiceDigito == 4) {
      validarSenha();
	}
  }
}
 
void validarSenha() {
  bool correta = true;
  for (int i = 0; i < 4; i++) {
	if (tentativaUsuario[i] != SENHA_MESTRA[i]) correta = false;
  }
 
  lcd.clear();
  if (correta) {
    lcd.print("Senha Correta!");
    estadoAtual = ABERTO;
    digitalWrite(4, LOW);
    digitalWrite(5, HIGH); //LED verde
    emitirSom(1500, 500);
    lcd.clear();
    lcd.print("Abrindo o Cofre");
    tranca.write(90); //Aciona o servo
  } else {
    erros++;
    lcd.print("Senha Incorreta");
    lcd.setCursor(0, 1);
    lcd.print("Erros: ");
    lcd.print(erros);
	
    emitirSom(400, 1000);
    delay(2000);
	
	if (erros >= 3) {
      estadoAtual = ALERTA;
      digitalWrite(4, LOW);
      digitalWrite(3, HIGH); //LED amarelo
      lcd.clear();
  	lcd.print("====SISTEMA=====");
      lcd.setCursor(0, 1);
      lcd.print("===BLOQUEADO====");
  	
      for(int i=0; i<3; i++) {
        emitirSom(450, 1000);
  	}
	} else {
      indiceDigito = 0;
      exibirMenuInicial();
	}
  }
}
