# TESTE
TESTE
/*
  NAVE vs METEOROS + BOSS  —  Arduino UNO + 4x MAX7219 (Wokwi)

  Ligações:
    Botão -> pino 2 (outro lado no GND)
    DIN -> 11 | CS -> 10 | CLK -> 13
  Hardware: 4x MAX7219 tipo PAROLA (MD_MAX72XX::PAROLA_HW), SPI por hardware.

  COMO JOGAR
    - Tela inicial: SPACE X + botão PLAY. 1 clique abre a escolha de dificuldade.
    - Escolha EASY ou HARD: 1 clique alterna, SEGURAR o botão ~1 s confirma.
    - A nave atira sozinha. Cada clique no botão muda a direção.
    - 3 vidas. Meteoros 3x3 aguentam 3 tiros.
    - Power-ups: tiro duplo, escudo, fantasma.
    - Depois de 45 s o BOSS aparece. Derrote-o para vencer.

  EASTER EGGS
    1) Desviar de 5 tiros seguidos do boss e depois acertar 4 tiros seguidos nele
       -> o boss EXPLODE (animação) e você vence na hora.
    2) Clicar 12 vezes rápido em até 3 s (em qualquer momento da partida)
       -> a tela pisca 2 s com vários corações e a nave vira um CORAÇÃO
          soltando faíscas por 5 s (1 vez por partida).
*/

#include <MD_MAX72xx.h>
#include <SPI.h>

// ================= CONFIGURAÇÕES =================
#define PIN_CS    10
#define PIN_BTN   2
#define NUM_MOD   4
#define BRILHO    3        // 0..15

#define HARDWARE_TYPE MD_MAX72XX::PAROLA_HW
MD_MAX72XX mx = MD_MAX72XX(HARDWARE_TYPE, PIN_CS, NUM_MOD);

#define NAVE_NO_PRIMEIRO_MODULO 1
#define ESPELHAR_X              0

#define W 8
#define H 32

#define VIDAS_INICIAIS      3
#define TEMPO_PARA_BOSS     45000UL
#define METEORO_HP          3
#define BOSS_HP             40
#define CHANCE_POWER        12
#define DURACAO_TIRO_DUPLO  10000UL
#define DURACAO_FANTASMA    6000UL

#define TICK_NAVE        110
#define INTERVALO_TIRO   380
#define TICK_TIRO        55
#define TICK_POWER       170
#define TICK_TIRO_BOSS   120
#define BOSS_FOGO_NORMAL 1900UL
#define BOSS_FOGO_FURIA  1300UL

#define BOSS_ATRASO_LINHAS 4       // ataque duplo: o 2º tiro sai 4 LEDs atrás do 1º (menor = mais difícil)

// alerta do boss: 3 caveiras piscando rápido, depois PREPARE (diagonal) / TO / DIE (diagonal)
#define ALERTA_ROSTO_MS  2000UL
#define ALERTA_MSG_MS    1200UL    // cada página da mensagem
#define ALERTA_PAGINAS   3         // PREPARE / TO / DIE
#define ALERTA_TOTAL_MS  (ALERTA_ROSTO_MS + ALERTA_PAGINAS * ALERTA_MSG_MS)

// easter eggs
#define EGG_DESVIOS_NECESSARIOS 5      // tiros do boss desviados em sequência
#define EGG_ACERTOS_NECESSARIOS 4      // tiros seguidos no boss depois disso
#define EGG_CLIQUES             12     // cliques rápidos
#define EGG_JANELA_MS           3000UL // ...dentro deste intervalo
#define EGG_CORACAO_MS          5000UL // duração do coração
#define EGG_PISCA_MS            2000UL // tela de corações antes da transformação

#define NAVE_Y 29

// ================= TELA =================
void flushTela() { mx.update(); }
void limpar()    { mx.clear(); }

void px(int x, int y) {
  if (x < 0 || x >= W || y < 0 || y >= H) return;
  int col = NAVE_NO_PRIMEIRO_MODULO ? (H - 1 - y) : y;
  int row = ESPELHAR_X ? x : (7 - x);
  mx.setPoint(row, col, true);
}

bool pxLigado(int x, int y) {
  if (x < 0 || x >= W || y < 0 || y >= H) return false;
  int col = NAVE_NO_PRIMEIRO_MODULO ? (H - 1 - y) : y;
  int row = ESPELHAR_X ? x : (7 - x);
  return mx.getPoint(row, col);
}

void preencherLinha(int y) { for (int x = 0; x < W; x++) px(x, y); }

void iniciarMax() {
  mx.begin();
  mx.control(MD_MAX72XX::INTENSITY, BRILHO);
  mx.control(MD_MAX72XX::UPDATE, MD_MAX72XX::OFF);
  limpar();
  flushTela();
}

// ================= SPRITES =================
#define NUM_FORMAS 6
const uint8_t meteoroForma[NUM_FORMAS][3] = {
  { 0b110, 0b111, 0b011 },
  { 0b011, 0b111, 0b110 },
  { 0b110, 0b111, 0b111 },
  { 0b111, 0b111, 0b010 },
  { 0b010, 0b111, 0b110 },
  { 0b111, 0b011, 0b011 }
};

const uint8_t powerAnel[5] = { 0b01110, 0b10001, 0b10001, 0b10001, 0b01110 };
const uint8_t powerIcone[3][3] = {
  { 0b101, 0b101, 0b101 },
  { 0b111, 0b111, 0b010 },
  { 0b010, 0b111, 0b101 }
};

// Boss: caveira com chifres + mandíbula solta (7 colunas x 8 linhas)
const uint8_t bossForma[8] = {
  0b1000001,
  0b1100011,
  0b0111110,
  0b0101010,
  0b0111110,
  0b0011100,
  0b0000000,
  0b0011100
};
#define BOSS_W 7
#define BOSS_ALTURA 8

// Caveira do alerta: 8 colunas (par) -> fica perfeitamente centralizada nas 8 colunas da tela
const uint8_t caveiraAlerta[8] = {
  0b10000001,
  0b11000011,
  0b01111110,
  0b01011010,
  0b01111110,
  0b00111100,
  0b00000000,
  0b00111100
};

// ================= ESTADO DO JOGO =================
enum Estado { TITULO, SELECAO, JOGANDO, ALERTA_BOSS, MOSTRA_VIDAS };
Estado estado = TITULO;

struct Tiro    { int8_t x, y; bool on; };
struct Meteoro { int8_t x, y; bool on; uint8_t forma; int8_t hp; uint32_t flashAte; };
struct Power   { int8_t x, y; uint8_t tipo; bool on; };

#define MAX_TIROS    12
#define MAX_METEOROS 6
#define MAX_POWERS   3
#define MAX_TIROS_BOSS 8

Tiro    tiros[MAX_TIROS];
Tiro    tirosBoss[MAX_TIROS_BOSS];
Meteoro meteoros[MAX_METEOROS];
Power   powers[MAX_POWERS];

int8_t   naveX, dir;
uint8_t  vidas, powersColetados;
bool     escudo;
uint32_t duploAte, fantasmaAte, invAte;
uint16_t pontos;
uint32_t tempoVivo, tUltTempo;

bool     bossAtivo, bossMorto;
int8_t   bossX, bossDir;
int16_t  bossHp;
uint32_t bossFlashAte;
uint8_t  bossPadrao;
bool     tiroAtrasado = false;
int8_t   tiroAtrasadoX = 0;      // lado do 2º tiro do ataque duplo: 0 = canhão esquerdo, 1 = direito
uint8_t  bossJitter = 100;       // variação aleatória (%) do intervalo entre tiros do boss
int8_t   bossUltPadrao = -1;
uint32_t tiroAtrasadoEm = 0;

// --- easter eggs ---
uint8_t  desviosBoss = 0;        // tiros do boss desviados em sequência
bool     eggArmado = false;      // já desviou de 5 -> agora vale acertar 4 seguidos
uint8_t  acertosSeguidos = 0;
bool     bossEgg = false;        // boss morreu pelo easter egg (explosão)
uint32_t cliqueT[EGG_CLIQUES];   // [0] = clique mais recente
uint8_t  cliqueN = 0;
bool     coracaoUsado = false;
uint32_t coracaoAte = 0;

uint32_t tNave, tDisparo, tTiro, tMet, tSpawn, tPower, tBossMove, tBossFire, tBossTiro;
uint32_t alertaAte, alertaIni, vidasAte;

uint16_t dificuldadeX10 = 10;
bool     selHard = false, selPressionando = false, selNivelAnt = HIGH;
uint32_t selPressIni = 0, selEdge = 0;

// ---------- botão ----------
bool ultBtn = HIGH;
uint32_t tBtn = 0;
bool lerBotao() {
  bool l = digitalRead(PIN_BTN);
  if (l != ultBtn && millis() - tBtn > 25) {
    tBtn = millis();
    ultBtn = l;
    if (l == LOW) return true;
  }
  return false;
}

void desenharCoracao(int x, int y) {
  px(x, y); px(x + 2, y);
  px(x, y + 1); px(x + 1, y + 1); px(x + 2, y + 1);
  px(x + 1, y + 2);
}

// tela cheia de corações piscando por 2 s; depois a nave vira coração (5 s)
void animCoracoes() {
  uint32_t t0 = millis();
  uint8_t f = 0;
  while (millis() - t0 < EGG_PISCA_MS) {
    limpar();
    if ((f & 1) == 0) {                                   // quadros pares: corações, ímpares: apagado
      for (uint8_t k = 0; k < 8; k++) {
        int x0 = ((k + f / 2) & 1) ? 1 : 0;               // linhas alternam de lado (e trocam a cada piscada)
        desenharCoracao(x0, k * 4);
        desenharCoracao(x0 + 4, k * 4);
      }
    }
    flushTela();
    delay(100);
    f++;
  }
  limpar(); flushTela();

  // o tempo da animação não conta contra o jogador
  uint32_t agora = millis();
  uint32_t dur = agora - t0;
  alertaIni += dur; alertaAte += dur; vidasAte += dur;
  invAte += dur; duploAte += duploAte ? dur : 0; fantasmaAte += fantasmaAte ? dur : 0;
  tiroAtrasadoEm += dur;
  tUltTempo = agora;
  tNave = tDisparo = tTiro = tMet = tSpawn = tPower = tBossMove = tBossFire = tBossTiro = agora;
  coracaoAte = agora + EGG_CORACAO_MS;
}

// 12 cliques em até 3 s -> corações na tela + nave coração (1x por partida)
void registrarClique(uint32_t now) {
  for (int8_t i = EGG_CLIQUES - 1; i > 0; i--) cliqueT[i] = cliqueT[i - 1];
  cliqueT[0] = now;
  if (cliqueN < EGG_CLIQUES) cliqueN++;
  if (cliqueN >= EGG_CLIQUES && !coracaoUsado && (now - cliqueT[EGG_CLIQUES - 1]) <= EGG_JANELA_MS) {
    coracaoUsado = true;
    cliqueN = 0;
    Serial.println(F("EASTER EGG: nave coracao!"));
    animCoracoes();
  }
}

void resetEggBoss() {
  desviosBoss = 0;
  eggArmado = false;
  acertosSeguidos = 0;
}

// ---------- utilidades ----------
uint32_t escalar(uint32_t ms) { return ms * 10UL / dificuldadeX10; }

bool naveOcupa(int x, int y) {
  int dx = x - naveX;
  if (y == NAVE_Y)     return dx == 0;
  if (y == NAVE_Y + 1) return abs(dx) <= 1;
  if (y == NAVE_Y + 2) return abs(dx) == 1;
  if (escudo && y == NAVE_Y - 1) return abs(dx) <= 1;
  return false;
}

bool pixelDoBoss(int x, int y) {
  if (!bossAtivo) return false;
  if (y < 0 || y >= BOSS_ALTURA) return false;
  int c = x - bossX;
  if (c < 0 || c >= BOSS_W) return false;
  return bossForma[y] & (1 << (BOSS_W - 1 - c));
}

bool pixelDoMeteoro(uint8_t j, int x, int y) {
  int dx = x - meteoros[j].x;
  int dy = y - meteoros[j].y;
  if (dx < 0 || dx > 2 || dy < 0 || dy > 2) return false;
  return meteoroForma[meteoros[j].forma][dy] & (1 << (2 - dx));
}

void desenharForma(uint8_t forma, int x, int y) {
  for (uint8_t r = 0; r < 3; r++)
    for (uint8_t c = 0; c < 3; c++)
      if (meteoroForma[forma][r] & (1 << (2 - c))) px(x + c, y + r);
}

void pxNave(int x, int y, bool fantasma, uint32_t now) {
  if (fantasma && (((x + y) + (now / 150)) & 1)) return;
  px(x, y);
}

void novoTiro(int x, int y) {
  if (x < 0 || x >= W) return;
  for (uint8_t i = 0; i < MAX_TIROS; i++) {
    if (!tiros[i].on) { tiros[i] = { (int8_t)x, (int8_t)y, true }; return; }
  }
}

void novoTiroBoss(int x, int y) {
  if (x < 0 || x >= W) return;
  for (uint8_t i = 0; i < MAX_TIROS_BOSS; i++) {
    if (!tirosBoss[i].on) { tirosBoss[i] = { (int8_t)x, (int8_t)y, true }; return; }
  }
}

void disparar(uint32_t now) {
  if (now < duploAte) {
    novoTiro(naveX - 1, NAVE_Y - 1);
    novoTiro(naveX + 1, NAVE_Y - 1);
  } else {
    novoTiro(naveX, NAVE_Y - 1);
  }
}

void talvezPower(int mx, int my) {
  if (bossAtivo) return;
  if (random(1000) >= (long)(CHANCE_POWER * 100L / dificuldadeX10)) return;
  for (uint8_t i = 0; i < MAX_POWERS; i++) {
    if (!powers[i].on) {
      powers[i] = { (int8_t)constrain(mx - 1, 0, 3), (int8_t)(my - 1), (uint8_t)random(3), true };
      return;
    }
  }
}

void spawnMeteoro() {
  int8_t x = random(0, 6);
  for (uint8_t i = 0; i < MAX_METEOROS; i++)
    if (meteoros[i].on && meteoros[i].y < 4 && abs(meteoros[i].x - x) < 3) return;
  for (uint8_t i = 0; i < MAX_METEOROS; i++) {
    if (!meteoros[i].on) {
      meteoros[i] = { x, -2, true, (uint8_t)random(NUM_FORMAS), METEORO_HP, 0 };
      return;
    }
  }
}

// ---------- dano / power-ups ----------
void dano(uint32_t now) {
  if (now < invAte || now < fantasmaAte) return;
  if (escudo) {
    escudo = false;
    invAte = now + 600;
    Serial.println(F("Escudo absorveu o dano!"));
    return;
  }
  if (vidas > 0) vidas--;
  invAte = now + 1500;
  Serial.print(F("Dano! Vidas: ")); Serial.println(vidas);
  if (vidas > 0) {
    estado = MOSTRA_VIDAS;
    vidasAte = now + 900;
  }
}

void iniciarAlertaBoss(uint32_t now) {
  for (uint8_t i = 0; i < MAX_METEOROS; i++) meteoros[i].on = false;
  for (uint8_t i = 0; i < MAX_POWERS; i++) powers[i].on = false;
  for (uint8_t i = 0; i < MAX_TIROS; i++) tiros[i].on = false;
  estado = ALERTA_BOSS;
  alertaIni = now;
  alertaAte = now + ALERTA_TOTAL_MS;
  Serial.println(F("!!! BOSS CHEGANDO !!!"));
}

void coletarPower(uint8_t tipo, uint32_t now) {
  if (tipo == 0) {
    duploAte = now + DURACAO_TIRO_DUPLO;
    Serial.println(F("Power-up: TIRO DUPLO"));
  } else if (tipo == 1) {
    escudo = true;
    Serial.println(F("Power-up: ESCUDO"));
  } else {
    fantasmaAte = now + DURACAO_FANTASMA;
    Serial.println(F("Power-up: FANTASMA"));
  }
  powersColetados++;
  Serial.print(F("Power-ups coletados: ")); Serial.println(powersColetados);
}

bool powerBate(uint8_t i) {
  for (uint8_t r = 0; r < 5; r++)
    for (uint8_t c = 0; c < 5; c++) {
      bool on = powerAnel[r] & (1 << (4 - c));
      if (!on && r >= 1 && r <= 3 && c >= 1 && c <= 3)
        on = powerIcone[powers[i].tipo][r - 1] & (1 << (3 - c));
      if (on && naveOcupa(powers[i].x + c, powers[i].y + r)) return true;
    }
  return false;
}

// ---------- colisões ----------
void colisoesTiros(uint32_t now) {
  for (uint8_t i = 0; i < MAX_TIROS; i++) {
    if (!tiros[i].on) continue;
    for (uint8_t j = 0; j < MAX_METEOROS; j++) {
      if (!meteoros[j].on) continue;
      if (pixelDoMeteoro(j, tiros[i].x, tiros[i].y)) {
        tiros[i].on = false;
        meteoros[j].hp--;
        if (meteoros[j].hp <= 0) {
          meteoros[j].on = false;
          pontos++;
          talvezPower(meteoros[j].x, meteoros[j].y);
        } else {
          meteoros[j].flashAte = now + 90;
        }
        break;
      }
    }
    if (tiros[i].on && pixelDoBoss(tiros[i].x, tiros[i].y)) {
      tiros[i].on = false;
      bossHp--;
      bossFlashAte = now + 90;
      if (eggArmado) {                                   // EASTER EGG: 4 acertos seguidos
        acertosSeguidos++;
        if (acertosSeguidos >= EGG_ACERTOS_NECESSARIOS) {
          bossEgg = true;
          bossMorto = true;
          Serial.println(F("EASTER EGG: boss explodiu!"));
        }
      }
      if (bossHp <= 0) bossMorto = true;
    }
  }
}

void colisoesNave(uint32_t now) {
  bool fantasma = now < fantasmaAte;

  if (!fantasma) {
    for (uint8_t j = 0; j < MAX_METEOROS; j++) {
      if (!meteoros[j].on) continue;
      bool bateu = false;
      for (uint8_t dy = 0; dy < 3 && !bateu; dy++)
        for (uint8_t dx = 0; dx < 3 && !bateu; dx++)
          if (pixelDoMeteoro(j, meteoros[j].x + dx, meteoros[j].y + dy) &&
              naveOcupa(meteoros[j].x + dx, meteoros[j].y + dy)) bateu = true;
      if (bateu) {
        meteoros[j].on = false;
        dano(now);
      }
    }
    for (uint8_t i = 0; i < MAX_TIROS_BOSS; i++) {
      if (tirosBoss[i].on && naveOcupa(tirosBoss[i].x, tirosBoss[i].y)) {
        tirosBoss[i].on = false;
        resetEggBoss();                                  // levou tiro: zera a sequência do easter egg
        dano(now);
      }
    }
  }
  for (uint8_t i = 0; i < MAX_POWERS; i++) {
    if (!powers[i].on) continue;
    if (powerBate(i)) {
      powers[i].on = false;
      coletarPower(powers[i].tipo, now);
    }
  }
}

// ---------- novo jogo ----------
void novoJogo(uint32_t now) {
  for (uint8_t i = 0; i < MAX_TIROS; i++) tiros[i].on = false;
  for (uint8_t i = 0; i < MAX_TIROS_BOSS; i++) tirosBoss[i].on = false;
  for (uint8_t i = 0; i < MAX_METEOROS; i++) meteoros[i].on = false;
  for (uint8_t i = 0; i < MAX_POWERS; i++) powers[i].on = false;
  naveX = 3;
  dir = 0;
  vidas = VIDAS_INICIAIS;
  powersColetados = 0;
  escudo = false;
  duploAte = 0;
  fantasmaAte = 0;
  invAte = 0;
  pontos = 0;
  tempoVivo = 0;
  tUltTempo = now;
  bossAtivo = false;
  bossMorto = false;
  bossFlashAte = 0;
  bossPadrao = 0;
  tiroAtrasado = false;
  resetEggBoss();
  bossEgg = false;
  cliqueN = 0;
  coracaoUsado = false;
  coracaoAte = 0;
  tNave = tDisparo = tTiro = tMet = tSpawn = tPower = tBossMove = tBossFire = tBossTiro = now;
  Serial.println(F("=== JOGO INICIADO ==="));
}

// ---------- atualização do jogo ----------
void atualizarJogo(uint32_t now, bool click) {
  if (click) dir = (dir == 0) ? -1 : -dir;

  uint32_t delta = now - tUltTempo;
  tUltTempo = now;
  if (delta < 500) tempoVivo += delta;

  if (now - tNave >= TICK_NAVE) {
    tNave = now;
    naveX = constrain(naveX + dir, 1, 6);
  }

  if (now - tDisparo >= INTERVALO_TIRO) {
    tDisparo = now;
    disparar(now);
  }

  // tiros do jogador
  if (now - tTiro >= TICK_TIRO) {
    tTiro = now;
    for (uint8_t i = 0; i < MAX_TIROS; i++) {
      if (!tiros[i].on) continue;
      tiros[i].y--;
      if (tiros[i].y < 0) {
        tiros[i].on = false;
        if (bossAtivo && eggArmado) acertosSeguidos = 0;   // errou o tiro: sequência quebrada
      }
    }
    colisoesTiros(now);
  }

  if (!bossAtivo) {
    uint32_t intSpawn = escalar(max(700L, 1400L - (long)pontos * 15L));
    if (now - tSpawn >= intSpawn) {
      tSpawn = now;
      spawnMeteoro();
    }
  }
  uint32_t intMet = escalar(max(150L, 300L - (long)pontos * 5L));
  if (now - tMet >= intMet) {
    tMet = now;
    for (uint8_t i = 0; i < MAX_METEOROS; i++) {
      if (!meteoros[i].on) continue;
      meteoros[i].y++;
      if (meteoros[i].y > H - 1) meteoros[i].on = false;
    }
    colisoesTiros(now);
  }

  if (now - tPower >= TICK_POWER) {
    tPower = now;
    for (uint8_t i = 0; i < MAX_POWERS; i++) {
      if (!powers[i].on) continue;
      powers[i].y++;
      if (powers[i].y > H - 1) powers[i].on = false;
    }
  }

  // BOSS
  if (bossAtivo) {
    bool furioso = bossHp <= (BOSS_HP * 4) / 10;

    if (now - tBossMove >= escalar(furioso ? 200UL : 280UL)) {
      tBossMove = now;
      bossX += bossDir;
      if (bossX <= 0) { bossX = 0; bossDir = 1; }
      if (bossX >= W - BOSS_W) { bossX = W - BOSS_W; bossDir = -1; }
    }

    if (now - tBossFire >= escalar(furioso ? BOSS_FOGO_FURIA : BOSS_FOGO_NORMAL) * bossJitter / 100) {
      tBossFire = now;
      bossJitter = random(70, 131);                       // próximo tiro vem 70%..130% do intervalo normal
      int yTiro = BOSS_ALTURA;

      // padrão aleatório (esq 25% | dir 25% | mirado 25% | duplo 25%), evita repetir o mesmo 2x seguidas
      int8_t padrao;
      do {
        int r = random(20);
        padrao = (r < 5) ? 0 : (r < 10) ? 1 : (r < 15) ? 3 : 2;
      } while (padrao == bossUltPadrao && random(3) != 0);
      bossUltPadrao = padrao;

      switch (padrao) {
        case 0: novoTiroBoss(bossX + 1, yTiro); break;
        case 1: novoTiroBoss(bossX + 5, yTiro); break;
        case 2: {                                       // ataque duplo: lado inicial sorteado, 2º tiro bem depois
          bool atrasaEsq = random(2);
          novoTiroBoss(atrasaEsq ? bossX + 5 : bossX + 1, yTiro);
          tiroAtrasadoX = atrasaEsq ? 0 : 1;                // sai do outro canhão, na posição do boss nesse momento
          tiroAtrasadoEm = now + escalar(TICK_TIRO_BOSS) * BOSS_ATRASO_LINHAS;
          tiroAtrasado = true;
          break;
        }
        case 3: novoTiroBoss(constrain(naveX, bossX + 1, bossX + 5), yTiro); break;  // mirado
      }
      bossPadrao++;
    }

    if (tiroAtrasado && now >= tiroAtrasadoEm) {
      tiroAtrasado = false;
      novoTiroBoss(tiroAtrasadoX ? bossX + 5 : bossX + 1, BOSS_ALTURA);
    }

    if (now - tBossTiro >= escalar(TICK_TIRO_BOSS)) {
      tBossTiro = now;
      for (uint8_t i = 0; i < MAX_TIROS_BOSS; i++) {
        if (!tirosBoss[i].on) continue;
        tirosBoss[i].y++;
        if (tirosBoss[i].y > H - 1) {
          tirosBoss[i].on = false;                         // passou direto: tiro desviado
          if (!eggArmado) {
            desviosBoss++;
            if (desviosBoss >= EGG_DESVIOS_NECESSARIOS) {
              eggArmado = true;
              acertosSeguidos = 0;
              Serial.println(F("EASTER EGG armado: acerte 4 tiros seguidos no boss!"));
            }
          }
        }
      }
    }
  }

  colisoesNave(now);

  if (!bossAtivo && estado == JOGANDO && tempoVivo >= TEMPO_PARA_BOSS) iniciarAlertaBoss(now);
}

// ---------- desenho ----------
void desenharJogo(uint32_t now) {
  limpar();

  bool fant = now < fantasmaAte;
  bool coracao = now < coracaoAte;
  if (now >= invAte || ((now / 90) & 1)) {
    if (coracao) {
      // nave em forma de coração:  # . #
      //                            # # #
      //                            . # .
      pxNave(naveX - 1, NAVE_Y, fant, now);     pxNave(naveX + 1, NAVE_Y, fant, now);
      pxNave(naveX - 1, NAVE_Y + 1, fant, now); pxNave(naveX, NAVE_Y + 1, fant, now); pxNave(naveX + 1, NAVE_Y + 1, fant, now);
      pxNave(naveX, NAVE_Y + 2, fant, now);
    } else {
      pxNave(naveX, NAVE_Y, fant, now);
      pxNave(naveX - 1, NAVE_Y + 1, fant, now); pxNave(naveX, NAVE_Y + 1, fant, now); pxNave(naveX + 1, NAVE_Y + 1, fant, now);
      pxNave(naveX - 1, NAVE_Y + 2, fant, now); pxNave(naveX + 1, NAVE_Y + 2, fant, now);
      if (now < duploAte) { pxNave(naveX - 1, NAVE_Y, fant, now); pxNave(naveX + 1, NAVE_Y, fant, now); }
    }
    if (escudo) { px(naveX - 1, NAVE_Y - 1); px(naveX, NAVE_Y - 1); px(naveX + 1, NAVE_Y - 1); }
  }

  // faíscas ao redor do coração
  if (coracao) {
    for (uint8_t k = 0; k < 5; k++) {
      int sx = naveX + (int)random(-3, 4);
      int sy = NAVE_Y + 1 + (int)random(-4, 4);
      px(sx, sy);
    }
  }

  for (uint8_t i = 0; i < MAX_TIROS; i++)
    if (tiros[i].on) px(tiros[i].x, tiros[i].y);

  for (uint8_t i = 0; i < MAX_METEOROS; i++)
    if (meteoros[i].on && (now >= meteoros[i].flashAte || ((now / 40) & 1)))
      desenharForma(meteoros[i].forma, meteoros[i].x, meteoros[i].y);

  for (uint8_t i = 0; i < MAX_POWERS; i++) {
    if (!powers[i].on) continue;
    bool piscar = (now / 150) & 1;
    for (uint8_t r = 0; r < 5; r++)
      for (uint8_t c = 0; c < 5; c++) {
        bool on = powerAnel[r] & (1 << (4 - c));
        if (!on && piscar && r >= 1 && r <= 3 && c >= 1 && c <= 3)
          on = powerIcone[powers[i].tipo][r - 1] & (1 << (3 - c));
        if (on) px(powers[i].x + c, powers[i].y + r);
      }
  }

  if (bossAtivo && (now >= bossFlashAte || ((now / 40) & 1))) {
    for (uint8_t r = 0; r < BOSS_ALTURA; r++)
      for (uint8_t c = 0; c < BOSS_W; c++)
        if (bossForma[r] & (1 << (BOSS_W - 1 - c))) px(bossX + c, r);
  }

  for (uint8_t i = 0; i < MAX_TIROS_BOSS; i++)
    if (tirosBoss[i].on) px(tirosBoss[i].x, tirosBoss[i].y);

  flushTela();
}

void desenharVidas() {
  limpar();
  for (uint8_t i = 0; i < VIDAS_INICIAIS; i++) {
    int x = i * 3;
    int y = 26;
    if (i < vidas) { px(x, y); px(x + 1, y); px(x, y + 1); px(x + 1, y + 1); }
    else           { px(x, y); }
  }
  flushTela();
}

// ---------- fonte 3x5 ----------
// (agora com a letra X)
const char letrasFonte[] = "ACDEGHIJLMOPRSTVXY!";
const uint8_t fonte[][5] PROGMEM = {
  { 0b010, 0b101, 0b111, 0b101, 0b101 },  // A
  { 0b011, 0b100, 0b100, 0b100, 0b011 },  // C
  { 0b110, 0b101, 0b101, 0b101, 0b110 },  // D
  { 0b111, 0b100, 0b110, 0b100, 0b111 },  // E
  { 0b011, 0b100, 0b101, 0b101, 0b011 },  // G
  { 0b101, 0b101, 0b111, 0b101, 0b101 },  // H
  { 0b111, 0b010, 0b010, 0b010, 0b111 },  // I
  { 0b001, 0b001, 0b001, 0b101, 0b010 },  // J
  { 0b100, 0b100, 0b100, 0b100, 0b111 },  // L
  { 0b101, 0b111, 0b111, 0b101, 0b101 },  // M
  { 0b010, 0b101, 0b101, 0b101, 0b010 },  // O
  { 0b110, 0b101, 0b110, 0b100, 0b100 },  // P
  { 0b110, 0b101, 0b110, 0b101, 0b101 },  // R
  { 0b011, 0b100, 0b010, 0b001, 0b110 },  // S
  { 0b111, 0b010, 0b010, 0b010, 0b010 },  // T
  { 0b101, 0b101, 0b101, 0b101, 0b010 },  // V
  { 0b101, 0b101, 0b010, 0b101, 0b101 },  // X
  { 0b101, 0b101, 0b010, 0b010, 0b010 },  // Y
  { 0b010, 0b010, 0b010, 0b000, 0b010 }   // !
};

void desenharLetra(char c, int x, int y) {
  if (c == 0) return;
  const char* p = strchr(letrasFonte, c);
  if (!p) return;
  uint8_t idx = p - letrasFonte;
  for (uint8_t r = 0; r < 5; r++) {
    uint8_t linha = pgm_read_byte(&fonte[idx][r]);
    for (uint8_t k = 0; k < 3; k++)
      if (linha & (4 >> k)) px(x + k, y + r);
  }
}

void desenharPar(const char* s, int y) {
  if (s[1]) { desenharLetra(s[0], 0, y); desenharLetra(s[1], 5, y); }
  else      { desenharLetra(s[0], 3, y); }
}

// ---------- fundo de estrelas ----------
const uint8_t  estX[8]   = { 0, 5, 2, 7, 3, 6, 1, 4 };
const uint16_t estVel[8] = { 140, 200, 110, 260, 170, 230, 125, 190 };
const uint8_t  estOff[8] = { 0, 9, 17, 5, 22, 13, 27, 3 };

bool pxLivre(int x, int y) {
  for (int dx = -1; dx <= 1; dx++)
    for (int dy = -1; dy <= 1; dy++)
      if (pxLigado(x + dx, y + dy)) return false;
  return true;
}

void desenharEstrelas(uint32_t now) {
  for (uint8_t i = 0; i < 8; i++) {
    int y = (int)((now / estVel[i] + estOff[i]) % H);
    if (pxLivre(estX[i], y)) px(estX[i], y);
  }
}

// ---------- tela de título ----------
const uint8_t naveGrande[7] = {
  0b00100,
  0b00100,
  0b01110,
  0b01010,
  0b11111,
  0b10101,
  0b10001
};

void telaTitulo(uint32_t now) {
  limpar();
  if ((now % 5200) < 3000) {
    // logo "SPACE X" em 3 linhas: SP / AC / EX
    desenharPar("SP", 1);
    desenharPar("AC", 7);
    desenharPar("EX", 13);

    for (uint8_t r = 0; r < 7; r++)
      for (uint8_t c = 0; c < 5; c++)
        if (naveGrande[r] & (1 << (4 - c))) px(1 + c, 24 + r);
    if ((now / 100) & 1) { px(1, 31); px(5, 31); }
    else                 { px(3, 31); }
    px(3, 23 - (int)((now / 70) % 5));
  } else {
    bool cheio = (now / 350) & 1;
    for (uint8_t i = 0; i < 11; i++) {
      uint8_t w = (i <= 5) ? i + 1 : 11 - i;
      int y = 3 + i;
      if (cheio) { for (uint8_t x = 1; x <= w; x++) px(x, y); }
      else       { px(1, y); px(w, y); }
    }
    desenharPar("PL", 17);
    desenharPar("AY", 23);
  }
  desenharEstrelas(now);
  flushTela();
}

// ---------- tela de escolha EASY / HARD ----------
void telaSelecao(uint32_t now) {
  limpar();
  desenharPar("EA", 2);
  desenharPar("SY", 8);
  desenharPar("HA", 19);
  desenharPar("RD", 25);
  if (!selPressionando || ((now / 80) & 1)) {
    preencherLinha(selHard ? 17 : 0);
    preencherLinha(selHard ? 31 : 14);
  }
  desenharEstrelas(now);
  flushTela();
}

bool atualizarSelecao(uint32_t now) {
  bool nivel = digitalRead(PIN_BTN);
  if (now - selEdge > 30) {
    if (selNivelAnt == HIGH && nivel == LOW) {
      selPressionando = true;
      selPressIni = now;
      selEdge = now;
    } else if (selNivelAnt == LOW && nivel == HIGH) {
      selEdge = now;
      if (selPressionando) { selHard = !selHard; selPressionando = false; }
    }
    selNivelAnt = nivel;
  }
  if (selPressionando && nivel == LOW && now - selPressIni >= 900) {
    selPressionando = false;
    return true;
  }
  return false;
}

// ---------- GAME OVER / VICTORY ----------
const char* const fimGameOver[4] = { "GA", "ME", "OV", "ER" };
const char* const fimVitoria[4]  = { "VI", "CT", "OR", "Y!" };
const uint8_t fimY[4] = { 4, 10, 16, 22 };

void desenharExplosao(int cx, int cy, int r) {
  int interno = (r > 1) ? (r - 2) * (r - 2) : -1;
  for (int dx = -r; dx <= r; dx++)
    for (int dy = -r; dy <= r; dy++) {
      int d2 = dx * dx + dy * dy;
      if (d2 <= r * r && d2 > interno) px(cx + dx, cy + dy);
    }
}

// EASTER EGG: o boss explode (treme, explosões pequenas, onda grande e flash)
void animExplosaoBoss() {
  int cx = bossX + BOSS_W / 2;
  for (int f = 0; f < 18; f++) {
    limpar();
    // boss tremendo / piscando
    if (f < 9 && (f & 1) == 0) {
      int tremor = (f & 2) ? 1 : 0;
      for (uint8_t r = 0; r < BOSS_ALTURA; r++)
        for (uint8_t c = 0; c < BOSS_W; c++)
          if (bossForma[r] & (1 << (BOSS_W - 1 - c))) px(bossX + c + tremor, r);
    }
    // explosões pequenas aleatórias sobre o boss
    for (uint8_t k = 0; k < 2; k++)
      desenharExplosao(bossX + (int)random(BOSS_W), (int)random(8), (int)random(1, 3));
    // onda grande de explosão
    if (f >= 6) desenharExplosao(cx, 4, f - 5);
    flushTela();
    delay(80);
  }
  // flash final
  for (uint8_t i = 0; i < 3; i++) {
    limpar();
    for (int y = 0; y < H; y++) preencherLinha(y);
    flushTela();
    delay(110);
    limpar(); flushTela();
    delay(90);
  }
}

void animFim(bool vitoria) {
  const char* const* linhas = vitoria ? fimVitoria : fimGameOver;
  uint32_t inicio = millis();

  if (!vitoria) {
    for (int r = 0; r <= 8; r++) {
      limpar();
      desenharExplosao(naveX, NAVE_Y + 1, r);
      flushTela();
      delay(70);
    }
    limpar(); flushTela();
    delay(150);
  }

  uint32_t t0 = millis();
  for (;;) {
    uint32_t t = millis() - t0;
    if (t > 4800) break;
    limpar();

    bool pousou = t >= 1300;
    bool aceso = !pousou || ((t - 1300) % 500) < 340;
    if (aceso) {
      for (uint8_t k = 0; k < 4; k++) {
        int dt = (int)t - k * 320;
        if (dt < 0) continue;
        int subir = (dt >= 240) ? 0 : (240 - dt) * 12 / 240;
        desenharPar(linhas[k], fimY[k] - subir);
      }
    }

    if (vitoria) {
      for (uint8_t k = 0; k < 5; k++) {
        int gx = random(W), gy = random(H);
        if (pxLivre(gx, gy)) px(gx, gy);
      }
    } else {
      desenharEstrelas(millis());
    }
    flushTela();
    delay(40);
    if (millis() - inicio > 1500 && digitalRead(PIN_BTN) == LOW) break;
  }
  limpar(); flushTela();
}

void animGameOver() {
  Serial.print(F("GAME OVER! Meteoros destruidos: ")); Serial.println(pontos);
  animFim(false);
}

void animVitoria() {
  Serial.println(F("*** VITORIA! BOSS DESTRUIDO! ***"));
  if (bossEgg) animExplosaoBoss();
  animFim(true);
}

// palavra em diagonal (de cima-esquerda para baixo-direita), fonte 3x5, uma letra por degrau
void desenharDiagonal(const char* s, int y0, int passoY) {
  uint8_t n = strlen(s);
  for (uint8_t i = 0; i < n; i++) {
    int x = (n > 1) ? 1 + (i * 3) / (n - 1) : 3;  // colunas 1..4: o conjunto ocupa 1..6 (margem de 1 LED nos dois lados)
    desenharLetra(s[i], x, y0 + i * passoY);
  }
}

// mini-fonte 3x3 (só as letras de PREPARE) para caber a palavra inteira na tela
const char letrasMini[] = "PREA";
const uint8_t fonteMini[4][3] = {
  { 0b110, 0b110, 0b100 },  // P
  { 0b110, 0b110, 0b101 },  // R
  { 0b111, 0b110, 0b111 },  // E
  { 0b010, 0b111, 0b101 }   // A
};

// "PREPARE" inteiro em diagonal: 7 letras 3x3, degraus de 4/5 linhas (1 a 2 LEDs de vão), colunas 1..6
void desenharPrepare() {
  const char* palavra = "PREPARE";
  for (uint8_t i = 0; i < 7; i++) {
    const char* q = strchr(letrasMini, palavra[i]);
    uint8_t idx = q - letrasMini;
    int x = 1 + (i * 3) / 6;
    int y = 1 + (i * 9) / 2;                         // 1,5,10,14,19,23,28
    for (uint8_t r = 0; r < 3; r++)
      for (uint8_t c = 0; c < 3; c++)
        if (fonteMini[idx][r] & (4 >> c)) px(x + c, y + r);
  }
}

// alerta do boss: 3 caveiras centralizadas piscando rápido (2 s),
// depois 3 páginas: PREPARE (diagonal) / TO / DIE (diagonal)
void telaAlertaBoss(uint32_t t) {
  limpar();
  if (t < ALERTA_ROSTO_MS) {
    if (((t / 110) & 1) == 0) {
      for (uint8_t k = 0; k < 3; k++)
        for (uint8_t r = 0; r < 8; r++)
          for (uint8_t c = 0; c < 8; c++)
            if (caveiraAlerta[r] & (1 << (7 - c))) px(c, 2 + k * 10 + r);
    }
  } else {
    uint32_t m = t - ALERTA_ROSTO_MS;
    uint8_t pagina = m / ALERTA_MSG_MS;
    uint32_t local = m % ALERTA_MSG_MS;
    if ((local % 600) < 420) {                       // pisca 2x em cada página
      if (pagina == 0)      desenharPrepare();
      else if (pagina == 1) desenharPar("TO", 13);
      else                  desenharDiagonal("DIE", 6, 7);
    }
  }
  flushTela();
}

void voltarAoTitulo() {
  estado = TITULO;
  delay(400);
  ultBtn = digitalRead(PIN_BTN);
  tBtn = millis();
}

// ================= SETUP / LOOP =================
void setup() {
  Serial.begin(115200);
  pinMode(PIN_CS, OUTPUT);
  digitalWrite(PIN_CS, HIGH);
  pinMode(PIN_BTN, INPUT_PULLUP);
  randomSeed(analogRead(A0));
  iniciarMax();
  Serial.println(F("Aperte o botao para jogar."));
}

void loop() {
  uint32_t now = millis();
  bool click = lerBotao();

  // easter egg do coração: vale em qualquer momento da partida
  if (click && (estado == JOGANDO || estado == ALERTA_BOSS || estado == MOSTRA_VIDAS))
    registrarClique(now);

  switch (estado) {
    case TITULO:
      telaTitulo(now);
      if (click) {
        selHard = false;
        selPressionando = false;
        selNivelAnt = digitalRead(PIN_BTN);
        selEdge = now;
        estado = SELECAO;
      }
      break;

    case SELECAO:
      telaSelecao(now);
      if (atualizarSelecao(now)) {
        dificuldadeX10 = selHard ? 18 : 10;
        randomSeed(micros());
        novoJogo(now);
        estado = JOGANDO;
        Serial.println(selHard ? F("Dificuldade: HARD") : F("Dificuldade: EASY"));
      }
      break;

    case JOGANDO:
      atualizarJogo(now, click);
      if (vidas == 0) {
        animGameOver();
        voltarAoTitulo();
      } else if (bossMorto) {
        animVitoria();
        voltarAoTitulo();
      } else if (estado == JOGANDO) {
        desenharJogo(now);
      }
      break;

    case ALERTA_BOSS:
      if (click) dir = (dir == 0) ? -1 : -dir;
      telaAlertaBoss(now - alertaIni);
      if (now >= alertaAte) {
        bossAtivo = true;
        bossX = 0;
        bossDir = 1;
        bossHp = BOSS_HP;
        bossPadrao = 0;
        bossJitter = 100;
        bossUltPadrao = -1;
        tiroAtrasado = false;
        resetEggBoss();
        tNave = tDisparo = tTiro = tMet = tSpawn = tPower = tBossMove = tBossFire = tBossTiro = now;
        tBossFire = now + 600;
        estado = JOGANDO;
      }
      break;

    case MOSTRA_VIDAS:
      if (click) dir = (dir == 0) ? -1 : -dir;
      desenharVidas();
      if (now >= vidasAte) {
        tNave = tDisparo = tTiro = tMet = tSpawn = tPower = tBossMove = tBossFire = tBossTiro = now;
        estado = JOGANDO;
      }
      break;
  }
}
