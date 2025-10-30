import pygame
import math
import random
from pygame.locals import *

# Inicializa Pygame
pygame.init()
WIDTH, HEIGHT = 1200, 800
screen = pygame.display.set_mode((WIDTH, HEIGHT), DOUBLEBUF)
pygame.display.set_caption("Zumbis de Angola - O Último Sobrevivente")
clock = pygame.time.Clock()

# Cores
AZUL = (0, 0, 255)  # Jogador
VERMELHO = (255, 0, 0)  # Zumbis
VERDE = (0, 255, 0)  # Suprimentos
PRETO = (0, 0, 0)
BRANCO = (255, 255, 255)
MARROM = (139, 69, 19)  # Terreno

# Configurações do Mundo (Angola ~ 1.246.000 km², escalado para grid 50x50)
MAP_SIZE = 50  # Grid simples para mundo aberto
PLAYER_SPEED = 0.5
ZOMBIE_SPEED = 0.2
NUM_ZOMBIES = 15
NUM_SUPRIMENTOS = 10

# Cidades reais de Angola (coordenadas aproximadas em grid: x=longitude escalada, z=latitude escalada)
# Base: Luanda (0° lat, 0° lon) como centro. Escala: 1 unidade ~ 25.000 km (simplificado)
CIDADES = {
    "Luanda": (0, 0),  # Capital, costa norte
    "Lobito": (-2, 1),  # Porto sul de Luanda
    "Benguela": (-3, 2),  # Cidade costeira
    "Huambo": (-5, -3),  # Planaltos centrais, perto de Mount Moco
    "Lubango": (-6, 4),  # Sul, perto de Namib
    "Cabinda": (1, -2),  # Enclave norte
    "Malange": (-4, -1),  # Centro
    "Uige": (0.5, -1.5),  # Norte
    "Namibe": (-7, 5),  # Deserto sul
    "Kalandula Falls": (-3.5, -2.5)  # Quedas d'água (landmark)
}

# Posições iniciais aleatórias no mapa
player_pos = [25.0, 1.0, 25.0]  # [x, y(altura), z] - Começa em "Luanda"
zombies = [[random.uniform(0, MAP_SIZE), 1.0, random.uniform(0, MAP_SIZE)] for _ in range(NUM_ZOMBIES)]
suprimentos = [[random.uniform(0, MAP_SIZE), 1.0, random.uniform(0, MAP_SIZE)] for _ in range(NUM_SUPRIMENTOS)]
pontos = 0
font = pygame.font.SysFont('Arial', 24)

# Função para renderizar ponto 3D simples (projeção perspectiva)
def project(point, cam_pos, cam_yaw, cam_pitch):
    # Vetor relativo
    dx = point[0] - cam_pos[0]
    dz = point[2] - cam_pos[2]
    # Rotaciona pela yaw (horizontal)
    dx_rot = dx * math.cos(cam_yaw) + dz * math.sin(cam_yaw)
    dz_rot = -dx * math.sin(cam_yaw) + dz * math.cos(cam_yaw)
    # Pitch simples (eleva y)
    dy = point[1] - cam_pos[1]
    dy_rot = dy * math.cos(cam_pitch) - dz_rot * math.sin(cam_pitch)
    dz_rot = dy * math.sin(cam_pitch) + dz_rot * math.cos(cam_pitch)
    
    if dz_rot < 0.1:  # Evita divisão por zero
        return None
    # Projeção perspectiva
    factor = 300 / dz_rot
    x = WIDTH / 2 + dx_rot * factor
    y = HEIGHT / 2 - dy_rot * factor
    size = int(5 * factor)
    return (int(x), int(y), max(size, 1))

# Desenha o mapa no chão (grid simples)
def draw_map(screen, cam_pos):
    for x in range(0, MAP_SIZE + 1):
        for z in range(0, MAP_SIZE + 1):
            proj = project([x, 0, z], cam_pos, 0, 0)
            if proj:
                pygame.draw.circle(screen, MARROM, (proj[0], proj[1]), 1)

# Loop principal do jogo
running = True
cam_yaw = 0  # Rotação horizontal
cam_pitch = 0  # Rotação vertical
mouse_sens = 0.002

# Captura mouse relativo
pygame.mouse.set_visible(False)
pygame.event.set_grab(True)

while running:
    dt = clock.tick(60) / 1000.0  # Tempo real
    screen.fill(PRETO)
    
    for event in pygame.event.get():
        if event.type == QUIT or (event.type == KEYDOWN and event.key == K_ESCAPE):
            running = False
        if event.type == MOUSEMOTION:
            dx, dy = event.rel
            cam_yaw += dx * mouse_sens
            cam_pitch -= dy * mouse_sens  # Inverte Y
            cam_pitch = max(-math.pi/2, min(math.pi/2, cam_pitch))
    
    keys = pygame.key.get_pressed()
    # Movimento FPS (WASD)
    forward = (keys[K_w] - keys[K_s]) * PLAYER_SPEED * dt
    strafe = (keys[K_d] - keys[K_a]) * PLAYER_SPEED * dt
    
    # Direção de movimento baseada em yaw
    player_pos[0] += math.cos(cam_yaw) * forward + math.sin(cam_yaw) * strafe
    player_pos[2] += math.sin(cam_yaw) * forward - math.cos(cam_yaw) * strafe
    
    # Limites do mapa (mundo aberto de Angola)
    player_pos[0] = max(0, min(MAP_SIZE, player_pos[0]))
    player_pos[2] = max(0, min(MAP_SIZE, player_pos[2]))
    
    # Desenha o terreno (chão do mapa)
    draw_map(screen, player_pos)
    
    # Desenha cidades como "bases" (círculos maiores)
    for nome, (x, z) in CIDADES.items():
        point = [x + 25, 0.5, z + 25]  # Offset para centro do grid
        proj = project(point, player_pos, cam_yaw, cam_pitch)
        if proj:
            size = proj[2] * 2
            pygame.draw.circle(screen, BRANCO, (proj[0], proj[1]), size)
            # Label simples (apenas se perto)
            if proj[2] > 10:
                text = font.render(nome[:3], True, BRANCO)  # Abreviação
                screen.blit(text, (proj[0] + size, proj[1]))
    
    # Atualiza e desenha zumbis (IA simples: perseguem jogador)
    for zombie in zombies[:]:
        # Movimento em direção ao jogador
        dx = player_pos[0] - zombie[0]
        dz = player_pos[2] - zombie[2]
        dist = math.sqrt(dx**2 + dz**2)
        if dist > 0:
            zombie[0] += (dx / dist) * ZOMBIE_SPEED * dt
            zombie[2] += (dz / dist) * ZOMBIE_SPEED * dt
        
        proj = project(zombie, player_pos, cam_yaw, cam_pitch)
        if proj:
            size = proj[2]
            pygame.draw.circle(screen, VERMELHO, (proj[0], proj[1]), size)
        
        # Colisão com jogador (morte)
        if dist < 0.5:
            print(f"Você foi pego! Pontos finais: {pontos}")
            running = False
    
    # Suprimentos (colete para pontos)
    for sup in suprimentos[:]:
        proj = project(sup, player_pos, cam_yaw, cam_pitch)
        if proj:
            size = proj[2]
            pygame.draw.circle(screen, VERDE, (proj[0], proj[1]), size)
        
        # Colisão
        dx = player_pos[0] - sup[0]
        dz = player_pos[2] - sup[2]
        dist = math.sqrt(dx**2 + dz**2)
        if dist < 0.5:
            suprimentos.remove(sup)
            pontos += 1
            # Spawn novo suprimento aleatório
            suprimentos.append([random.uniform(0, MAP_SIZE), 1.0, random.uniform(0, MAP_SIZE)])
    
    # HUD: Pontos, posição, cidade mais próxima
    text = font.render(f"Pontos: {pontos} | Pos: ({player_pos[0]:.1f}, {player_pos[2]:.1f})", True, BRANCO)
    screen.blit(text, (10, 10))
    
    # Cidade mais próxima
    min_dist = float('inf')
    cidade_prox = "Explorando"
    for nome, (x, z) in CIDADES.items():
        point = [x + 25, 0, z + 25]
        dx = player_pos[0] - point[0]
        dz = player_pos[2] - point[2]
        dist = math.sqrt(dx**2 + dz**2)
        if dist < min_dist:
            min_dist = dist
            cidade_prox = nome
    text_cidade = font.render(f"Próximo: {cidade_prox}", True, BRANCO)
    screen.blit(text_cidade, (10, 40))
    
    # Instruções
    instr = [
        "WASD: Mover | Mouse: Olhar",
        "Verde: Colete suprimentos",
        "Vermelho: Fuja dos zumbis!",
        "Explore Angola: Luanda, Huambo, etc."
    ]
    for i, line in enumerate(instr):
        text = font.render(line, True, BRANCO)
        screen.blit(text, (10, HEIGHT - 100 + i*25))
    
    pygame.display.flip()

pygame.quit()
print("Fim de jogo. Sobreviventes: 1 (você, até o próximo apocalipse)!")
