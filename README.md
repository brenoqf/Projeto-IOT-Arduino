# Projeto IOT - 5º Semestre

## 1. Relatório Técnico

### Descrição do Projeto
Este projeto consiste em um sistema de controle de acesso para um compartimento seguro, desenvolvido na plataforma **Arduino Uno**. O sistema gerencia a entrada de senhas via 4 botões, valida o acesso através de uma **Máquina de Estados Finita (FSM)** e fornece feedback em tempo real ao usuário por meio de componentes visuais (LCD e LEDs), sonoros (Buzzer) e mecânicos (Servo).

### Objetivos de Aprendizagem Alcançados
* **Integração de múltiplos periféricos:** Uso simultâneo de sensores e atuadores.
* **Lógica de Máquina de Estados:** Gerenciamento dos estados *Trancado*, *Aberto* e *Alerta*.
* **Interrupções de Hardware:** Implementação de um botão de Reset crítico via interrupção.
* **Interface de Usuário (UI):** Criação de mensagens dinâmicas e intuitivas no LCD 16x2.

### Componentes Utilizados
* Arduino Uno R3
* Display LCD 16x2
* Micro Servo Motor (Trava mecânica)
* Buzzer Piezoelétrico (Feedback sonoro)
* 3 LEDs (Verde, Vermelho e Amarelo)
* 5 Botões de Pressão (4 para senha, 1 para reset)
* Resistores (Diversos valores)

---

## 2. Diagrama de Conexão

| Componente | Pino Arduino | Tipo de Sinal |
| :--- | :--- | :--- |
| LCD (RS, E, D4-D7) | 13, 12, 11, 10, 9, 8 | Digital |
| Servomotor | 7 | Digital |
| Buzzer | 6 | Digital |
| LED Verde | 5 | Digital |
| LED Vermelho | 4 | Digital |
| LED Amarelo | 3 | Digital |
| Botão RESET | 2 | Interrupção |
| Botões de Senha | A0, A1, A2, A3 | Analógico |

---

## 3. Código Fonte

```cpp
#include <LiquidCrystal.h>
#include <Servo.h>

LiquidCrystal lcd(13, 12, 11, 10, 9, 8);

// Configurações da Senha
const int SENHA_MESTRA[] = {0, 1, 2, 3};
const int pinoReset = 2;

// Variáveis voláteis para garantir interrupção
volatile int tentativaUsuario[4];
volatile int indiceDigito = 0;
volatile int erros = 0;

// Pinos dos Botões
const int botoes[] = {A0, A1, A2, A3};

// Pino do Servo e Buzzer
const int pinoServo = 7;
const int pinoBuzzer = 6;
Servo tranca;

// Estados do Cofre
enum Estados { TRANCADO, ABERTO, ALERTA };
volatile Estados estadoAtual = TRANCADO;

void setup() {
  lcd.begin(16, 2);
  pinMode(pinoReset, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(pinoReset), resetarSistema, FALLING);

  for(int i = 0; i < 4; i++) {
    pinMode(botoes[i], INPUT);
  }
  
  pinMode(5, OUTPUT); // LED Verde
  pinMode(4, OUTPUT); // LED Vermelho
  pinMode(3, OUTPUT); // LED Amarelo
  
  tranca.attach(pinoServo);
  tranca.write(0);
  
  exibirMenuInicial();
}

void emitirSom(int frequencia, int duracao) {
  tone(pinoBuzzer, frequencia, duracao);
  delay(duracao);
  noTone(pinoBuzzer);
}

void resetarSistema() {
  erros = 0;
  indiceDigito = 0;
  estadoAtual = TRANCADO;
}

void loop() {
  if (estadoAtual == TRANCADO) {
    digitalWrite(5, LOW);  
    digitalWrite(3, LOW);  
    digitalWrite(4, HIGH); 
    tranca.write(0);
    
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
    digitalWrite(5, HIGH); 
    emitirSom(1500, 500);
    lcd.clear();
    lcd.print("Abrindo...");
    tranca.write(90);
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
      digitalWrite(3, HIGH);
      lcd.clear();
      lcd.print("BLOQUEADO");
    } else {
      indiceDigito = 0;
      exibirMenuInicial();
    }
  }
}
