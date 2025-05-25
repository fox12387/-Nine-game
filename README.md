# -Nine-game
import pygame
import random
import math
import sys
from pygame import mixer

# 初始化pygame
pygame.init()
mixer.init()

# 屏幕设置
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("无.人")

# 颜色定义
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
RED = (255, 0, 0)
GRAY = (100, 100, 100)
DARK_RED = (150, 0, 0)
PLAYER_COLOR = (70, 130, 180)  # 钢蓝色
SHADOW_COLOR = (40, 40, 40)    # 深灰色
BG_COLOR = (20, 20, 30)        # 深蓝黑色
YELLOW = (255, 255, 0)         # 教程提示颜色

# 游戏状态


class GameState:
    TITLE = 0
    TUTORIAL = 1
    PLAYING = 2
    GAME_OVER = 3
    WIN = 4

# 加载背景音乐


def load_music():
    try:
        mixer.music.load("音乐/back.flac")
        mixer.music.set_volume(0.5)  # 设置音量50%
    except:
        print("警告: 无法加载背景音乐")

# 创建简单图形替代图像资源


def create_player_image():
    surf = pygame.Surface((40, 60), pygame.SRCALPHA)
    # 身体
    pygame.draw.rect(surf, PLAYER_COLOR, (10, 10, 20, 40))
    # 头部
    pygame.draw.circle(surf, PLAYER_COLOR, (20, 10), 10)
    return surf


def create_shadow_image():
    surf = pygame.Surface((50, 70), pygame.SRCALPHA)
    # 扭曲的人形轮廓
    points = [
        (10, 10), (40, 5), (45, 50),
        (35, 65), (5, 60), (10, 10)
    ]
    pygame.draw.polygon(surf, SHADOW_COLOR, points)
    return surf


def create_mirror_image():
    surf = pygame.Surface((30, 40), pygame.SRCALPHA)
    pygame.draw.rect(surf, (200, 200, 255), (0, 0, 30, 40), 3)
    pygame.draw.line(surf, (200, 200, 255), (15, 0), (15, 40), 2)
    return surf


def create_flashlight_image():
    surf = pygame.Surface((40, 20), pygame.SRCALPHA)
    pygame.draw.rect(surf, (200, 200, 200), (0, 5, 30, 10))
    pygame.draw.circle(surf, (255, 255, 200), (35, 10), 8)
    return surf


def create_background(width, height, level):
    surf = pygame.Surface((width, height))
    surf.fill(BG_COLOR)

    # 根据关卡添加不同特征
    if level == 1:  # 商场
        # 绘制货架
        for i in range(3):
            pygame.draw.rect(surf, (30, 30, 40), (100 + i*200, 150, 100, 300))
        # 地面网格线
        for i in range(0, width, 50):
            pygame.draw.line(surf, (40, 40, 50), (i, 0), (i, height), 1)
        for i in range(0, height, 50):
            pygame.draw.line(surf, (40, 40, 50), (0, i), (width, i), 1)

    elif level == 2:  # 地铁
        # 地铁轨道
        pygame.draw.rect(surf, (50, 50, 60), (0, 400, width, 100))
        # 站台
        pygame.draw.rect(surf, (80, 80, 90), (0, 350, width, 50))
        # 柱子
        for i in range(4):
            pygame.draw.rect(surf, (70, 70, 80), (50 + i*200, 100, 20, 300))

    else:  # 公寓
        # 墙壁
        pygame.draw.rect(surf, (40, 35, 45), (100, 100, 600, 400))
        # 窗户
        pygame.draw.rect(surf, (30, 30, 50), (150, 150, 100, 100), 2)
        pygame.draw.rect(surf, (30, 30, 50), (300, 150, 100, 100), 2)
        pygame.draw.rect(surf, (30, 30, 50), (450, 150, 100, 100), 2)
        # 门
        pygame.draw.rect(surf, (60, 50, 40), (300, 300, 100, 200))

    return surf


# 加载资源
player_img = create_player_image()
shadow_img = create_shadow_image()
mirror_img = create_mirror_image()
flashlight_img = create_flashlight_image()
bg_mall = create_background(WIDTH, HEIGHT, 1)
bg_metro = create_background(WIDTH, HEIGHT, 2)
bg_apartment = create_background(WIDTH, HEIGHT, 3)
load_music()  # 加载背景音乐

# 玩家类


class Player(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.original_image = player_img
        self.image = self.original_image.copy()
        self.rect = self.image.get_rect()
        self.rect.center = (WIDTH // 4, HEIGHT // 2)
        self.speed = 5
        self.sanity = 100
        self.flashlight = True
        self.mirror = False
        self.infection = 0
        self.last_sanity_drain = 0
        self.flashlight_cooldown = 0
        self.flashlight_active = False
        self.flashlight_timer = 0

    def update(self):
        keys = pygame.key.get_pressed()
        if keys[pygame.K_w]:
            self.rect.y -= self.speed
        if keys[pygame.K_s]:
            self.rect.y += self.speed
        if keys[pygame.K_a]:
            self.rect.x -= self.speed
        if keys[pygame.K_d]:
            self.rect.x += self.speed

        # 边界检查
        self.rect.x = max(0, min(WIDTH - self.rect.width, self.rect.x))
        self.rect.y = max(0, min(HEIGHT - self.rect.height, self.rect.y))

        # 理智随时间缓慢下降
        now = pygame.time.get_ticks()
        if now - self.last_sanity_drain > 1000:
            self.sanity = max(0, self.sanity - 1)
            self.last_sanity_drain = now

        # 冷却时间更新
        if self.flashlight_cooldown > 0:
            self.flashlight_cooldown -= 1

        # 手电筒光效计时
        if self.flashlight_active:
            self.flashlight_timer -= 1
            if self.flashlight_timer <= 0:
                self.flashlight_active = False

    def use_flashlight(self):
        if self.flashlight and self.flashlight_cooldown == 0:
            self.flashlight_active = True
            self.flashlight_timer = 10  # 10帧的光效
            self.flashlight_cooldown = 30  # 0.5秒冷却(60FPS)
            return True
        return False

    def use_mirror(self):
        if self.mirror:
            self.mirror = False
            return True
        return False

# 影子敌人类


class Shadow(pygame.sprite.Sprite):
    def __init__(self, player, level):
        super().__init__()
        self.original_image = shadow_img
        self.image = self.original_image.copy()
        self.image.set_alpha(180)
        self.rect = self.image.get_rect()
        self.player = player
        self.level = level
        self.spawn_time = pygame.time.get_ticks()
        self.last_move = 0
        self.move_delay = max(100, 500 - level * 50)  # 随关卡增加移动速度
        self.health = 1  # 固定1点生命值，不再随关卡增加
        self.respawn()

    def respawn(self):
        # 在玩家视野外随机生成
        side = random.randint(0, 3)
        if side == 0:  # 上
            self.rect.x = random.randint(0, WIDTH)
            self.rect.y = -50
        elif side == 1:  # 右
            self.rect.x = WIDTH + 50
            self.rect.y = random.randint(0, HEIGHT)
        elif side == 2:  # 下
            self.rect.x = random.randint(0, WIDTH)
            self.rect.y = HEIGHT + 50
        else:  # 左
            self.rect.x = -50
            self.rect.y = random.randint(0, HEIGHT)

        self.spawn_time = pygame.time.get_ticks()

    def update(self):
        now = pygame.time.get_ticks()

        # 闪烁效果
        if random.random() < 0.1:
            self.image.set_alpha(random.randint(100, 200))
        else:
            self.image.set_alpha(180)

        # 向玩家移动
        if now - self.last_move > self.move_delay:
            dx = self.player.rect.centerx - self.rect.centerx
            dy = self.player.rect.centery - self.rect.centery
            dist = max(1, math.sqrt(dx*dx + dy*dy))

            # 速度基于距离和关卡难度
            speed = min(3 + self.level * 0.5, max(1, dist /
                        (50 - self.level * 3)))  # 减缓速度增长
            self.rect.x += int(dx / dist * speed)
            self.rect.y += int(dy / dist * speed)
            self.last_move = now

        # 如果碰到玩家，减少玩家理智
        if pygame.sprite.collide_rect(self, self.player):
            self.player.sanity = max(
                0, self.player.sanity - (3 + self.level // 2))  # 减少伤害
            self.player.infection = min(
                100, self.player.infection + (1 + self.level // 3))  # 减缓感染增长

        # 如果存在时间过长，重新生成
        if now - self.spawn_time > max(5000, 15000 - self.level * 500):  # 减缓重生时间缩短速度
            self.respawn()

# 物品类


class Item(pygame.sprite.Sprite):
    def __init__(self, item_type, x, y):
        super().__init__()
        self.item_type = item_type  # "mirror" 或 "battery"
        if item_type == "mirror":
            self.image = mirror_img
        else:
            self.image = flashlight_img
        self.rect = self.image.get_rect()
        self.rect.center = (x, y)

    def apply(self, player):
        if self.item_type == "mirror":
            player.mirror = True
            return "获得了镜子！按E键使用"
        else:
            player.flashlight = True
            player.sanity = min(100, player.sanity + 75)  # 增加电池恢复量
            return "找到了手电筒电池！理智恢复了75%"

# 游戏场景


class GameScene:
    def __init__(self):
        self.state = GameState.TITLE
        self.title_alpha = 0
        self.title_timer = 0
        self.player = Player()
        self.shadows = pygame.sprite.Group()
        self.items = pygame.sprite.Group()
        self.all_sprites = pygame.sprite.Group()
        self.game_over = False
        self.win = False
        self.message = ""
        self.message_time = 0
        self.current_bg = bg_mall
        self.level = 1
        self.max_level = 5
        self.tutorial_step = 0
        self.tutorial_messages = [
            "使用 WASD 键移动角色",
            "按 F 键使用手电筒击退影子",
            "收集镜子(按E使用)可以消灭所有影子",
            "保持理智，避免被完全感染",
            "祝你好运..."
        ]
        self.tutorial_timer = 0
        self.music_playing = False
        self.shadow_spawn_timer = 0
        self.shadow_spawn_delay = 3000
        self.encouragement_shown = False  # 鼓励语是否已显示
        self.encouragement_timer = 0  # 鼓励语显示计时器
        self.init_game()

    def init_game(self):
        self.state = GameState.TITLE
        self.title_alpha = 0
        self.title_timer = 0
        self.player = Player()
        self.shadows = pygame.sprite.Group()
        self.items = pygame.sprite.Group()
        self.all_sprites = pygame.sprite.Group(self.player)
        self.game_over = False
        self.win = False
        self.message = ""
        self.message_time = 0
        self.current_bg = bg_mall
        self.level = 1
        self.tutorial_step = 0
        self.tutorial_timer = 0
        self.shadow_spawn_timer = 0
        self.shadow_spawn_delay = max(1500, 3000 - self.level * 300)  # 调整生成间隔
        self.encouragement_shown = False
        self.encouragement_timer = 0
        self.spawn_shadows()
        self.spawn_items()
        self.play_music()

    def play_music(self):
        if not self.music_playing:
            try:
                mixer.music.play(-1)  # -1表示循环播放
                self.music_playing = True
            except:
                pass

    def spawn_shadows(self):
        # 调整影子生成数量，避免过多
        base_count = 2
        additional = min(3, self.level)  # 每关最多增加3个
        for _ in range(base_count + additional):
            shadow = Shadow(self.player, self.level)
            self.shadows.add(shadow)
            self.all_sprites.add(shadow)

    def spawn_items(self):
        # 确保每关至少有2个物品
        item_count = max(2, 4 - self.level // 2)
        for _ in range(item_count):
            x = random.randint(50, WIDTH - 50)
            y = random.randint(50, HEIGHT - 50)
            # 保持镜子生成概率稳定
            mirror_prob = 0.3
            item_type = "mirror" if random.random() < mirror_prob else "battery"
            item = Item(item_type, x, y)
            self.items.add(item)
            self.all_sprites.add(item)

    def update(self):
        if self.state == GameState.TITLE:
            self.title_alpha = min(255, self.title_alpha + 3)
            if self.title_alpha == 255:
                self.title_timer += 1
                if self.title_timer >= 180:  # 3秒后(60FPS)
                    self.state = GameState.TUTORIAL
                    self.encouragement_timer = pygame.time.get_ticks()  # 开始显示鼓励语
            return

        if self.state == GameState.TUTORIAL:
            # 显示鼓励语3秒后再开始教程
            if not self.encouragement_shown and pygame.time.get_ticks() - self.encouragement_timer < 3000:
                return
            self.encouragement_shown = True

            self.tutorial_timer += 1
            if self.tutorial_timer >= 180:  # 每个教程步骤显示3秒
                self.tutorial_step += 1
                self.tutorial_timer = 0
                if self.tutorial_step >= len(self.tutorial_messages):
                    self.state = GameState.PLAYING
            return

        if self.state in (GameState.GAME_OVER, GameState.WIN):
            return

        # 定时生成新的影子
        now = pygame.time.get_ticks()
        # 限制最大影子数量
        if now - self.shadow_spawn_timer > self.shadow_spawn_delay and len(self.shadows) < 3 + self.level:
            self.spawn_shadows()
            self.shadow_spawn_timer = now
            # 调整生成间隔
            self.shadow_spawn_delay = max(1000, 3000 - self.level * 300)

        self.all_sprites.update()

        # 检查物品碰撞
        for item in self.items:
            if pygame.sprite.collide_rect(item, self.player):
                self.message = item.apply(self.player)
                self.message_time = pygame.time.get_ticks()
                item.kill()

        # 检查手电筒是否照到影子
        if self.player.flashlight_active:
            for shadow in self.shadows:
                # 简单距离检测
                dx = shadow.rect.centerx - self.player.rect.centerx
                dy = shadow.rect.centery - self.player.rect.centery
                dist = math.sqrt(dx*dx + dy*dy)

                if dist < 200:
                    # 保持手电筒效率稳定
                    success_prob = 0.5
                    if random.random() < success_prob:
                        shadow.health -= 1
                        if shadow.health <= 0:
                            shadow.kill()
                            self.message = "手电筒击退了影子！"
                            self.message_time = pygame.time.get_ticks()

        # 检查游戏结束条件
        if self.player.sanity <= 0 or self.player.infection >= 100:
            self.state = GameState.GAME_OVER
        elif self.level == self.max_level and len(self.shadows) == 0:
            self.state = GameState.WIN

        # 检查是否进入下一关
        if len(self.shadows) == 0 and self.level < self.max_level:
            self.level += 1
            self.message = f"进入第 {self.level} 关！难度提升！"
            self.message_time = pygame.time.get_ticks()
            self.change_background()
            self.player.sanity = min(
                100, self.player.sanity + 30)  # 进入新关恢复更多理智
            self.shadows.empty()
            self.items.empty()
            self.spawn_shadows()
            self.spawn_items()

    def change_background(self):
        if self.level <= 2:
            self.current_bg = bg_mall
        elif self.level <= 4:
            self.current_bg = bg_metro
        else:
            self.current_bg = bg_apartment

    def draw(self):
        # 绘制背景
        screen.blit(self.current_bg, (0, 0))

        if self.state == GameState.TITLE:
            self.draw_title()
            return

        if self.state == GameState.TUTORIAL:
            # 显示鼓励语
            if not self.encouragement_shown or pygame.time.get_ticks() - self.encouragement_timer < 3000:
                self.draw_encouragement()
            else:
                self.draw_tutorial()
            return

        # 绘制所有精灵
        for sprite in self.all_sprites:
            screen.blit(sprite.image, sprite.rect)

        # 绘制手电筒光效
        if self.player.flashlight_active:
            # 创建光锥效果
            flashlight_cone = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
            pygame.draw.circle(flashlight_cone, (255, 255, 200, 80),
                               self.player.rect.center, 200)
            pygame.draw.polygon(flashlight_cone, (255, 255, 200, 80), [
                self.player.rect.center,
                (self.player.rect.centerx + 200, self.player.rect.centery - 100),
                (self.player.rect.centerx + 200, self.player.rect.centery + 100)
            ])
            screen.blit(flashlight_cone, (0, 0))

        # 绘制UI
        self.draw_ui()

        # 显示消息
        if pygame.time.get_ticks() - self.message_time < 3000:
            font = pygame.font.SysFont("simhei", 36)
            text = font.render(self.message, True, WHITE)
            screen.blit(text, (WIDTH // 2 - text.get_width() // 2, 50))

        # 游戏结束或胜利画面
        if self.state == GameState.GAME_OVER:
            self.draw_game_over()
        elif self.state == GameState.WIN:
            self.draw_win_screen()

    def draw_title(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 200))
        screen.blit(overlay, (0, 0))

        font_large = pygame.font.SysFont("simhei", 72)
        title1 = font_large.render("无.", True, (self.title_alpha, 0, 0))
        title2 = font_large.render("人", True, (self.title_alpha, 0, 0))

        screen.blit(title1, (WIDTH // 2 - title1.get_width() //
                    2 - 50, HEIGHT // 2 - 50))
        screen.blit(title2, (WIDTH // 2 - title2.get_width() //
                    2 + 50, HEIGHT // 2 - 50))

    def draw_encouragement(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 200))
        screen.blit(overlay, (0, 0))

        font = pygame.font.SysFont("simhei", 48)
        text = font.render("不要怕，孩子，你将战胜黑暗。", True, WHITE)
        screen.blit(text, (WIDTH // 2 - text.get_width() // 2, HEIGHT // 2))

    def draw_tutorial(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 200))
        screen.blit(overlay, (0, 0))

        font_large = pygame.font.SysFont("simhei", 48)
        font_small = pygame.font.SysFont("simhei", 24)

        title = font_large.render("新手教程", True, YELLOW)
        instruction = font_small.render(
            self.tutorial_messages[self.tutorial_step], True, WHITE)
        prompt = font_small.render("自动继续...", True, GRAY)

        screen.blit(
            title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 2 - 100))
        screen.blit(instruction, (WIDTH // 2 -
                    instruction.get_width() // 2, HEIGHT // 2))
        screen.blit(
            prompt, (WIDTH // 2 - prompt.get_width() // 2, HEIGHT // 2 + 150))

    def draw_ui(self):
        # 半透明背景
        ui_bg = pygame.Surface((WIDTH, 80), pygame.SRCALPHA)
        ui_bg.fill((0, 0, 0, 150))
        screen.blit(ui_bg, (0, 0))

        # 理智条
        pygame.draw.rect(screen, GRAY, (20, 20, 200, 20))
        pygame.draw.rect(screen, RED, (20, 20, 200 *
                         (self.player.sanity / 100), 20))

        # 感染条
        pygame.draw.rect(screen, GRAY, (20, 50, 200, 20))
        pygame.draw.rect(screen, DARK_RED, (20, 50, 200 *
                         (self.player.infection / 100), 20))

        # 显示文字
        font = pygame.font.SysFont("simhei", 24)
        sanity_text = font.render(f"理智: {self.player.sanity}%", True, WHITE)
        infection_text = font.render(
            f"感染: {self.player.infection}%", True, WHITE)
        screen.blit(sanity_text, (230, 20))
        screen.blit(infection_text, (230, 50))

        # 显示当前关卡
        level_bg = pygame.Surface((150, 40), pygame.SRCALPHA)
        level_bg.fill((0, 0, 0, 150))
        screen.blit(level_bg, (WIDTH - 160, 10))

        level_font = pygame.font.SysFont("simhei", 30)
        level_text = level_font.render(
            f"关卡 {self.level}/{self.max_level}", True, YELLOW)
        screen.blit(level_text, (WIDTH - 150, 15))

        # 显示控制提示
        controls = font.render("WASD移动 | F手电筒 | E镜子 | ESC退出", True, WHITE)
        screen.blit(
            controls, (WIDTH // 2 - controls.get_width() // 2, HEIGHT - 30))

    def draw_game_over(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 200))
        screen.blit(overlay, (0, 0))

        font_large = pygame.font.SysFont("simhei", 72)
        font_small = pygame.font.SysFont("simhei", 36)

        if self.player.infection >= 100:
            title = font_large.render("被同化了", True, RED)
            subtitle = font_small.render("你成为了'01'...", True, WHITE)
        else:
            title = font_large.render("理智崩溃", True, RED)
            subtitle = font_small.render("按R键重新开始", True, WHITE)

        screen.blit(
            title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 2 - 50))
        screen.blit(
            subtitle, (WIDTH // 2 - subtitle.get_width() // 2, HEIGHT // 2 + 20))

    def draw_win_screen(self):
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 200))
        screen.blit(overlay, (0, 0))

        font_large = pygame.font.SysFont("simhei", 72)
        font_small = pygame.font.SysFont("simhei", 36)

        title = font_large.render("暂时安全了", True, WHITE)
        subtitle1 = font_small.render("你击退了所有影子...暂时", True, WHITE)
        subtitle2 = font_small.render("但手臂上的'01'标记仍在扩散", True, RED)
        restart = font_small.render("按R键重新开始", True, WHITE)

        screen.blit(
            title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 2 - 80))
        screen.blit(subtitle1, (WIDTH // 2 -
                    subtitle1.get_width() // 2, HEIGHT // 2))
        screen.blit(subtitle2, (WIDTH // 2 -
                    subtitle2.get_width() // 2, HEIGHT // 2 + 40))
        screen.blit(
            restart, (WIDTH // 2 - restart.get_width() // 2, HEIGHT // 2 + 100))

    def handle_event(self, event):
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_f and self.state == GameState.PLAYING:
                self.player.use_flashlight()

            elif event.key == pygame.K_e and self.state == GameState.PLAYING:
                if self.player.use_mirror():
                    for shadow in self.shadows:
                        shadow.kill()
                    self.message = "镜子反射消灭了所有影子！"
                    self.message_time = pygame.time.get_ticks()

            elif event.key == pygame.K_r and self.state in (GameState.GAME_OVER, GameState.WIN):
                self.init_game()

            elif event.key == pygame.K_ESCAPE:
                pygame.quit()
                sys.exit()

# 主游戏循环


def main():
    clock = pygame.time.Clock()
    game = GameScene()

    running = True
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            game.handle_event(event)

        game.update()
        game.draw()

        pygame.display.flip()
        clock.tick(60)

    pygame.quit()


if __name__ == "__main__":
    main()
    ###########################################################################################################################
    #这是一个恐怖游戏，内有恐怖元素。01和影子是游戏的主角，努力生存吧。
#This is a terrifying game with elements of horror. 01 and Shadow are the main characters of the game. Work hard to survive.
