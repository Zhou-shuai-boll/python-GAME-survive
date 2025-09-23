import pygame
import random
import sys
import time
from datetime import datetime, timedelta

# 初始化pygame
pygame.init()

# 游戏窗口设置
WIDTH, HEIGHT = 1000, 800  # 从800x600扩大到1000x800
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("丧尸危机生存")

# 颜色定义
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BLUE = (0, 120, 255)
BROWN = (139, 69, 19)
GRAY = (100, 100, 100)
YELLOW = (255, 255, 0)
PURPLE = (128, 0, 128)
ORANGE = (255, 165, 0)
DARK_GREEN = (0, 100, 0)
LIGHT_BLUE = (173, 216, 230)

# 游戏状态
class GameState:
    def __init__(self):
        self.day = 1
        self.max_days = 7
        self.hour = 8  # 早上8点开始
        self.minute = 0  # 新增分钟计时
        self.health = 100
        self.hunger = 100
        self.thirst = 100
        self.fatigue = 0  # 新增疲劳系统，0-100
        self.food = 5
        self.water = 5
        self.medicine = 1
        self.weapon_level = 1
        self.location = "安全屋"
        self.game_over = False
        self.message = "欢迎来到丧尸危机生存游戏！你需要生存7天等待救援。"
        self.start_time = datetime.now()
        self.last_action_time = datetime.now()
        self.zombie_encounter_chance = 0.15  # 大幅降低基础遭遇概率（从0.3降到0.15）
        self.survived = False
        self.action_result = ""
        self.combat_result = ""
        self.show_combat_result = False
        self.combat_result_timer = 0
        self.exploring_location = ""  # 当前探索的地点
        self.encountered_zombie = False  # 是否遭遇了丧尸
        
    def add_time(self, hours, minutes=0, is_resting=False):
        """增加时间，处理天数变化"""
        # 先将时间转换为分钟进行计算
        total_minutes = self.hour * 60 + self.minute + hours * 60 + minutes
        self.hour = total_minutes // 60
        self.minute = total_minutes % 60
        
        # 处理天数变化
        if self.hour >= 24:
            self.day += 1
            self.hour = self.hour % 24
            self.message = f"第{self.day}天开始了..."
        
        # 每10分钟消耗资源（简化计算）
        time_passed_minutes = hours * 60 + minutes
        intervals = time_passed_minutes // 10  # 每10分钟一个间隔
        
        for _ in range(intervals):
            # 休息时消耗减半
            if is_resting:
                self.hunger = max(0, self.hunger - 1)  # 调整消耗速率
                self.thirst = max(0, self.thirst - 1.5)
                # 休息时不增加疲劳，反而减少
                self.fatigue = max(0, self.fatigue - 2)
            else:
                self.hunger = max(0, self.hunger - 2)
                self.thirst = max(0, self.thirst - 3)
                self.fatigue = min(100, self.fatigue + 1)
            
            # 如果饥饿或口渴为0，健康值下降
            if self.hunger == 0:
                self.health = max(0, self.health - 1)
            if self.thirst == 0:
                self.health = max(0, self.health - 2)
            
            # 如果疲劳值过高，健康值下降
            if self.fatigue >= 80:
                self.health = max(0, self.health - 1)
        
        # 检查游戏是否结束
        if self.health <= 0:
            self.game_over = True
            self.message = "你没能生存下来... 游戏结束！"
        elif self.day > self.max_days:
            self.game_over = True
            self.survived = True
            self.message = "恭喜！你成功生存了7天，救援已经到达！"
    
    def get_time_string(self):
        """获取时间字符串"""
        return f"第{self.day}天 {self.hour:02d}:{self.minute:02d}"
    
    def explore_location(self, location_name, zombie_chance, rewards, hours=2, minutes=0):
        """探索地点 - 修改后的逻辑"""
        self.location = location_name
        self.exploring_location = location_name
        self.add_time(hours, minutes)
        
        # 探索恢复5点疲劳值
        self.fatigue = max(0, self.fatigue - 5)
        
        # 大幅降低遭遇概率（原概率的50%）
        zombie_chance = zombie_chance * 0.5
        
        encounter_zombie = False
        # 探索时可能遇到丧尸
        if random.random() < zombie_chance:
            encounter_zombie = True
            self.encountered_zombie = True
            self.message = f"在{location_name}探索时遇到了丧尸！战斗结束后可以继续探索。"
        else:
            # 没有遇到丧尸，正常获得奖励
            self.encountered_zombie = False
            if "food" in rewards:
                self.food += rewards["food"]
            if "water" in rewards:
                self.water += rewards["water"]
            if "medicine" in rewards:
                self.medicine += rewards["medicine"]
            
            reward_text = f"在{location_name}探索了{hours}小时{minutes}分钟，找到了"
            rewards_found = []
            if rewards.get("food", 0) > 0:
                rewards_found.append(f"{rewards['food']}个食物")
            if rewards.get("water", 0) > 0:
                rewards_found.append(f"{rewards['water']}瓶水")
            if rewards.get("medicine", 0) > 0:
                rewards_found.append(f"{rewards['medicine']}个药品")
            
            if rewards_found:
                reward_text += "、" .join(rewards_found)
            else:
                reward_text += "一些有用的物品"
            
            self.message = reward_text
        
        return encounter_zombie
    
    def handle_combat(self, choice):
        """处理战斗选择 - 修改后的逻辑"""
        if choice == "fight":
            success_chance = 0.3 + (self.weapon_level * 0.1) + (self.health / 100 * 0.2)
            if random.random() < success_chance:
                self.weapon_level = min(3, self.weapon_level + 0.5)
                result = f"战斗胜利！武器经验增加至Lv.{self.weapon_level:.1f}"
                # 战斗胜利后可以获得部分探索奖励（减少的奖励）
                self.give_reduced_rewards()
            else:
                damage = random.randint(20, 40)
                self.health = max(0, self.health - damage)
                result = f"战斗失败！受到{damage}点伤害"
                # 战斗失败也能获得少量奖励
                self.give_minimal_rewards()
        else:  # flee
            flee_chance = 0.6 + (self.health / 100 * 0.2)
            if random.random() < flee_chance:
                result = "成功逃脱！"
                # 逃脱后可以获得部分探索奖励
                self.give_reduced_rewards()
            else:
                damage = random.randint(10, 25)
                self.health = max(0, self.health - damage)
                result = f"逃跑失败！受到{damage}点伤害"
                # 逃跑失败也能获得少量奖励
                self.give_minimal_rewards()
        
        self.combat_result = result
        self.show_combat_result = True
        self.combat_result_timer = pygame.time.get_ticks()
        
        # 战斗结束后不清除位置，保持在探索地点
        self.encountered_zombie = False
        self.message = f"战斗结束。你可以继续在{self.exploring_location}探索或返回安全屋。"
        
        return result
    
    def give_reduced_rewards(self):
        """给予减少的探索奖励"""
        rewards = self.get_location_rewards(self.exploring_location)
        if "food" in rewards:
            self.food += max(1, rewards["food"] // 2)
        if "water" in rewards:
            self.water += max(1, rewards["water"] // 2)
        if "medicine" in rewards:
            self.medicine += max(1, rewards["medicine"] // 2)
    
    def give_minimal_rewards(self):
        """给予最少的探索奖励"""
        rewards = self.get_location_rewards(self.exploring_location)
        if "food" in rewards:
            self.food += 1
        if "water" in rewards:
            self.water += 1
        if "medicine" in rewards and rewards["medicine"] > 0:
            self.medicine += 1
    
    def get_location_rewards(self, location_name):
        """获取地点的标准奖励"""
        rewards_map = {
            "周围环境": {"food": 1, "water": 1},
            "超市": {"food": 3, "water": 2},
            "医院": {"medicine": 2},
            "森林": {"water": 2}
        }
        return rewards_map.get(location_name, {})
    
    def continue_exploring(self):
        """继续探索当前地点"""
        if self.exploring_location and not self.encountered_zombie:
            rewards = self.get_location_rewards(self.exploring_location)
            # 继续探索时间较短，奖励也较少
            hours, minutes = 1, 0
            self.add_time(hours, minutes)
            
            # 继续探索的遭遇概率更低
            zombie_chance = self.zombie_encounter_chance * 0.3  # 原概率的30%
            
            if random.random() < zombie_chance:
                self.encountered_zombie = True
                self.message = f"继续在{self.exploring_location}探索时遇到了丧尸！"
                return "zombie"
            else:
                # 获得少量奖励
                if "food" in rewards:
                    self.food += max(1, rewards["food"] // 3)
                if "water" in rewards:
                    self.water += max(1, rewards["water"] // 3)
                if "medicine" in rewards:
                    self.medicine += max(1, rewards["medicine"] // 3)
                
                self.message = f"继续在{self.exploring_location}探索了{hours}小时，找到了一些额外资源。"
                return "success"
        return "no_exploration"
    
    def return_to_safehouse(self):
        """返回安全屋"""
        if self.location != "安全屋":
            # 返回路上遭遇丧尸的概率也很低
            return_chance = 0.1  # 降低返回路上遭遇丧尸的概率
            if random.random() < return_chance:
                self.message = "在返回安全屋的路上遇到了丧尸！"
                self.encountered_zombie = True
                return "zombie"
            else:
                self.location = "安全屋"
                self.exploring_location = ""
                self.add_time(0, 30)  # 返回消耗30分钟
                self.message = "已安全返回安全屋"
                return "safe"
        else:
            self.message = "你已经在安全屋了"
            return "already_safe"
    
    def rest(self, hours=3):
        """休息功能"""
        if self.location == "安全屋":
            # 安全屋休息效率翻倍
            old_fatigue = self.fatigue
            self.fatigue = max(0, self.fatigue - 40)  # 安全屋休息恢复更多
            fatigue_recovered = old_fatigue - self.fatigue
            self.add_time(hours, 0, is_resting=True)
            self.message = f"在安全屋休息{hours}小时，恢复了{fatigue_recovered}点疲劳值。"
        else:
            # 野外休息被袭击概率降低
            attack_chance = 0.15  # 降低野外休息被袭击概率
            if random.random() < attack_chance:
                self.message = "在野外休息时被丧尸袭击！"
                self.encountered_zombie = True
                return "zombie_attack"
            else:
                old_fatigue = self.fatigue
                self.fatigue = max(0, self.fatigue - 20)
                fatigue_recovered = old_fatigue - self.fatigue
                self.add_time(hours, 0, is_resting=True)
                self.message = f"在{self.location}休息{hours}小时，恢复了{fatigue_recovered}点疲劳值。"
        return "rested"
    
    def consume_resource(self, resource_type):
        """消耗资源，安全屋效果增强"""
        if resource_type == "food" and self.food > 0:
            self.food -= 1
            # 安全屋效果增强50%
            bonus = 20 if self.location == "安全屋" else 0
            self.hunger = min(100, self.hunger + 40 + bonus)
            self.add_time(0, 10)
            return f"进食消耗10分钟，饥饿感大大缓解。{'（安全屋效果增强）' if self.location == '安全屋' else ''}"
        
        elif resource_type == "water" and self.water > 0:
            self.water -= 1
            # 安全屋效果增强50%
            bonus = 25 if self.location == "安全屋" else 0
            self.thirst = min(100, self.thirst + 50 + bonus)
            self.add_time(0, 10)
            return f"喝水消耗10分钟，感觉好多了。{'（安全屋效果增强）' if self.location == '安全屋' else ''}"
        
        elif resource_type == "medicine" and self.medicine > 0:
            self.medicine -= 1
            # 安全屋效果增强50%
            bonus = 20 if self.location == "安全屋" else 0
            self.health = min(100, self.health + 40 + bonus)
            self.add_time(0, 10)
            return f"使用药品消耗10分钟，伤势好转了。{'（安全屋效果增强）' if self.location == '安全屋' else ''}"
        
        return f"没有{resource_type}了！"

# 创建游戏状态
game = GameState()

# 字体
try:
    font = pygame.font.SysFont('simhei', 24)
    title_font = pygame.font.SysFont('simhei', 36)
    small_font = pygame.font.SysFont('simhei', 20)
except:
    font = pygame.font.Font(None, 24)
    title_font = pygame.font.Font(None, 36)
    small_font = pygame.font.Font(None, 20)

# 加载图像（美化版）
def draw_safehouse():
    # 安全屋 - 更精致的绘制
    pygame.draw.rect(screen, BROWN, (400, 250, 200, 150))
    pygame.draw.polygon(screen, RED, [(400, 250), (500, 200), (600, 250)])
    pygame.draw.rect(screen, BLUE, (480, 300, 40, 60))
    # 添加细节
    pygame.draw.rect(screen, YELLOW, (490, 310, 20, 20))  # 窗户
    pygame.draw.circle(screen, YELLOW, (500, 330), 5)  # 门把手

def draw_supermarket():
    # 超市 - 更精致的绘制
    pygame.draw.rect(screen, LIGHT_BLUE, (400, 250, 200, 150))
    pygame.draw.rect(screen, WHITE, (420, 300, 40, 40))
    pygame.draw.rect(screen, WHITE, (540, 300, 40, 40))
    # 添加招牌
    pygame.draw.rect(screen, RED, (450, 230, 100, 20))
    market_text = small_font.render("超市", True, WHITE)
    screen.blit(market_text, (475, 232))

def draw_hospital():
    # 医院 - 更精致的绘制
    pygame.draw.rect(screen, WHITE, (400, 250, 200, 150))
    pygame.draw.rect(screen, RED, (450, 270, 100, 30))
    pygame.draw.circle(screen, RED, (500, 350), 30)
    # 添加十字标志
    pygame.draw.rect(screen, RED, (495, 320, 10, 60))
    pygame.draw.rect(screen, RED, (470, 345, 60, 10))

def draw_forest():
    # 森林 - 更精致的绘制
    for i in range(5):
        x = 350 + i * 100
        # 不同大小的树
        tree_height = random.randint(80, 120)
        pygame.draw.polygon(screen, DARK_GREEN, [
            (x, 400), 
            (x-20, 400-tree_height), 
            (x+20, 400-tree_height)
        ])
        # 树干
        pygame.draw.rect(screen, BROWN, (x-5, 400, 10, 30))

def draw_zombie():
    # 丧尸 - 更精致的绘制
    # 身体
    pygame.draw.circle(screen, GREEN, (500, 350), 30)
    # 眼睛
    pygame.draw.circle(screen, RED, (490, 340), 5)
    pygame.draw.circle(screen, RED, (510, 340), 5)
    # 嘴巴
    pygame.draw.arc(screen, RED, (485, 350, 30, 20), 0, 3.14, 3)
    # 手臂
    pygame.draw.line(screen, GREEN, (470, 350), (490, 360), 5)
    pygame.draw.line(screen, GREEN, (530, 350), (510, 360), 5)

def draw_background():
    """绘制美化背景"""
    # 天空
    screen.fill(LIGHT_BLUE)
    # 地面
    pygame.draw.rect(screen, DARK_GREEN, (0, 400, WIDTH, 400))
    # 云朵
    for i in range(3):
        x = 100 + i * 300
        pygame.draw.circle(screen, WHITE, (x, 100), 30)
        pygame.draw.circle(screen, WHITE, (x+20, 90), 25)
        pygame.draw.circle(screen, WHITE, (x+40, 100), 30)

# 按钮类
class Button:
    def __init__(self, x, y, width, height, text, color=BLUE, hover_color=(0, 150, 255), text_color=WHITE):
        self.rect = pygame.Rect(x, y, width, height)
        self.text = text
        self.color = color
        self.hover_color = hover_color
        self.text_color = text_color
        self.is_hovered = False
        
    def draw(self):
        color = self.hover_color if self.is_hovered else self.color
        pygame.draw.rect(screen, color, self.rect, border_radius=12)
        pygame.draw.rect(screen, WHITE, self.rect, 3, border_radius=12)
        
        text_surf = font.render(self.text, True, self.text_color)
        text_rect = text_surf.get_rect(center=self.rect.center)
        screen.blit(text_surf, text_rect)
        
    def check_hover(self, pos):
        self.is_hovered = self.rect.collidepoint(pos)
        
    def is_clicked(self, pos, event):
        if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
            return self.rect.collidepoint(pos)
        return False

# 创建按钮 - 重新布局避免冲突
buttons = {
    # 探索按钮放在左侧第一列
    "explore": Button(50, 550, 180, 50, "探索周围", BLUE, (0, 150, 255)),
    "supermarket": Button(50, 610, 180, 50, "去超市", GREEN, (50, 200, 50)),
    "hospital": Button(50, 670, 180, 50, "去医院", RED, (255, 100, 100)),
    "forest": Button(50, 730, 180, 50, "去森林", DARK_GREEN, (50, 150, 50)),
    
    # 资源管理按钮放在中间第二列
    "eat": Button(250, 550, 180, 50, "进食", ORANGE, (255, 200, 0)),
    "drink": Button(250, 610, 180, 50, "喝水", BLUE, (0, 180, 255)),
    "rest": Button(250, 670, 180, 50, "休息", PURPLE, (180, 0, 180)),
    "use_medicine": Button(250, 730, 180, 50, "使用药品", RED, (255, 100, 100)),
    
    # 行动按钮放在右侧第三列
    "return_safe": Button(450, 550, 180, 50, "返回安全屋", YELLOW, (255, 255, 100), BLACK),
    "continue_explore": Button(450, 610, 180, 50, "继续探索", GREEN, (50, 200, 50)),
    
    # 游戏结束按钮放在底部中央
    "continue": Button(400, 700, 200, 50, "继续游戏", GREEN, (50, 200, 50))
}

# 创建战斗按钮 - 调整位置适应新窗口
fight_button = Button(350, 500, 150, 50, "战斗", RED, (255, 100, 100))
flee_button = Button(550, 500, 150, 50, "逃跑", BLUE, (0, 150, 255))

# 游戏主循环
clock = pygame.time.Clock()
zombie_encounter = False
waiting_for_combat_choice = False

running = True
while running:
    current_time = pygame.time.get_ticks()
    mouse_pos = pygame.mouse.get_pos()
    
    # 检查战斗结果显示时间（显示3秒）
    if game.show_combat_result and current_time - game.combat_result_timer > 3000:
        game.show_combat_result = False
    
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
            
        if not game.game_over:
            if game.encountered_zombie and not game.show_combat_result:
                # 处理战斗按钮点击
                fight_button.check_hover(mouse_pos)
                flee_button.check_hover(mouse_pos)
                
                if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                    if fight_button.rect.collidepoint(mouse_pos):
                        game.handle_combat("fight")
                    elif flee_button.rect.collidepoint(mouse_pos):
                        game.handle_combat("flee")
            
            elif not game.encountered_zombie and not game.show_combat_result:
                # 处理正常游戏按钮点击
                for key, button in buttons.items():
                    button.check_hover(mouse_pos)
                    if button.is_clicked(mouse_pos, event):
                        if key == "explore":
                            result = game.explore_location("周围环境", 0.3, {"food": 1, "water": 1})
                            if result:  # 如果遇到丧尸
                                game.encountered_zombie = True
                        
                        elif key == "supermarket":
                            result = game.explore_location("超市", 0.5, {"food": 3, "water": 2})
                            if result:
                                game.encountered_zombie = True
                        
                        elif key == "hospital":
                            result = game.explore_location("医院", 0.4, {"medicine": 2})
                            if result:
                                game.encountered_zombie = True
                        
                        elif key == "forest":
                            result = game.explore_location("森林", 0.2, {"water": 2})
                            if result:
                                game.encountered_zombie = True
                        
                        elif key == "eat":
                            if game.food > 0:
                                result = game.consume_resource("food")
                                game.message = result
                            else:
                                game.message = "没有食物了！"
                        
                        elif key == "drink":
                            if game.water > 0:
                                result = game.consume_resource("water")
                                game.message = result
                            else:
                                game.message = "没有水了！"
                        
                        elif key == "rest":
                            result = game.rest()
                            if result == "zombie_attack":
                                game.encountered_zombie = True
                        
                        elif key == "use_medicine":
                            if game.medicine > 0:
                                result = game.consume_resource("medicine")
                                game.message = result
                            else:
                                game.message = "没有药品了！"
                        
                        elif key == "return_safe":
                            result = game.return_to_safehouse()
                            if result == "zombie":
                                game.encountered_zombie = True
                        
                        elif key == "continue_explore":
                            if game.exploring_location and game.location != "安全屋":  # 只有在探索地点才能继续探索
                                result = game.continue_exploring()
                                if result == "zombie":
                                    game.encountered_zombie = True
        
        else:  # 游戏结束后的继续按钮
            if buttons["continue"].is_clicked(mouse_pos, event):
                # 重置游戏
                game = GameState()
    
    # 绘制背景
    draw_background()
    
    # 绘制顶部状态栏（美化）
    pygame.draw.rect(screen, (50, 50, 50, 180), (0, 0, WIDTH, 120), border_radius=10)
    pygame.draw.rect(screen, (80, 80, 80), (0, 0, WIDTH, 120), 3, border_radius=10)
    
    time_text = title_font.render(game.get_time_string(), True, YELLOW)
    health_text = font.render(f"健康: {game.health}", True, GREEN if game.health > 50 else YELLOW if game.health > 20 else RED)
    hunger_text = font.render(f"饥饿: {game.hunger}", True, GREEN if game.hunger > 50 else YELLOW if game.hunger > 20 else RED)
    thirst_text = font.render(f"口渴: {game.thirst}", True, GREEN if game.thirst > 50 else YELLOW if game.thirst > 20 else RED)
    fatigue_text = font.render(f"疲劳: {game.fatigue}", True, GREEN if game.fatigue < 50 else YELLOW if game.fatigue < 80 else RED)
    
    screen.blit(time_text, (20, 20))
    screen.blit(health_text, (400, 20))
    screen.blit(hunger_text, (400, 60))
    screen.blit(thirst_text, (600, 20))
    screen.blit(fatigue_text, (600, 60))
    
    # 绘制资源栏（美化）
    pygame.draw.rect(screen, (50, 50, 50, 180), (0, 120, WIDTH, 60), border_radius=10)
    pygame.draw.rect(screen, (80, 80, 80), (0, 120, WIDTH, 60), 3, border_radius=10)
    
    food_text = font.render(f"食物: {game.food}", True, ORANGE)
    water_text = font.render(f"水: {game.water}", True, BLUE)
    medicine_text = font.render(f"药品: {game.medicine}", True, RED)
    weapon_text = font.render(f"武器等级: {game.weapon_level:.1f}", True, PURPLE)
    
    screen.blit(food_text, (20, 140))
    screen.blit(water_text, (200, 140))
    screen.blit(medicine_text, (380, 140))
    screen.blit(weapon_text, (560, 140))
    
    # 绘制当前位置
    location_bg = pygame.Rect(WIDTH//2 - 200, 190, 400, 50)
    pygame.draw.rect(screen, (50, 50, 50, 180), location_bg, border_radius=10)
    pygame.draw.rect(screen, (80, 80, 80), location_bg, 3, border_radius=10)
    
    location_text = title_font.render(f"当前位置: {game.location}", True, WHITE)
    screen.blit(location_text, (WIDTH//2 - location_text.get_width()//2, 200))
    
    # 绘制场景
    if game.location == "安全屋":
        draw_safehouse()
    elif game.location == "超市":
        draw_supermarket()
    elif game.location == "医院":
        draw_hospital()
    elif game.location == "森林":
        draw_forest()
    elif game.exploring_location:  # 探索场景
        pygame.draw.rect(screen, BROWN, (400, 250, 200, 150))
        pygame.draw.rect(screen, GRAY, (420, 270, 160, 110))
        # 添加探索标志
        pygame.draw.circle(screen, YELLOW, (500, 300), 20)
        search_text = small_font.render("探索中", True, BLACK)
        screen.blit(search_text, (485, 295))
    
    # 如果遇到丧尸且需要选择
    if game.encountered_zombie and not game.show_combat_result:
        draw_zombie()
        
        # 战斗提示框
        combat_bg = pygame.Rect(300, 150, 400, 120)
        pygame.draw.rect(screen, (50, 0, 0, 200), combat_bg, border_radius=15)
        pygame.draw.rect(screen, RED, combat_bg, 3, border_radius=15)
        
        combat_text = title_font.render("你遇到了丧尸！", True, RED)
        screen.blit(combat_text, (WIDTH//2 - combat_text.get_width()//2, 170))
        
        instruction_text = font.render("选择你的行动：", True, WHITE)
        screen.blit(instruction_text, (WIDTH//2 - instruction_text.get_width()//2, 220))
        
        # 绘制战斗按钮
        fight_button.check_hover(mouse_pos)
        flee_button.check_hover(mouse_pos)
        
        fight_button.draw()
        flee_button.draw()
    
    # 显示战斗结果
    elif game.show_combat_result:
        draw_zombie()
        
        # 结果提示框
        result_bg = pygame.Rect(300, 150, 400, 120)
        pygame.draw.rect(screen, (0, 0, 50, 200), result_bg, border_radius=15)
        pygame.draw.rect(screen, BLUE, result_bg, 3, border_radius=15)
        
        result_text = title_font.render("战斗结果", True, PURPLE)
        screen.blit(result_text, (WIDTH//2 - result_text.get_width()//2, 170))
        
        combat_result_text = font.render(game.combat_result, True, YELLOW)
        screen.blit(combat_result_text, (WIDTH//2 - combat_result_text.get_width()//2, 220))
        
        countdown = 3 - (current_time - game.combat_result_timer) // 1000
        countdown_text = small_font.render(f"{countdown}秒后继续", True, WHITE)
        screen.blit(countdown_text, (WIDTH//2 - countdown_text.get_width()//2, 250))
    
    else:
        # 绘制行动按钮
        for key, button in buttons.items():
            if key != "continue":
                # 只有在探索地点且不在安全屋时才显示继续探索按钮
                if key == "continue_explore":
                    if game.exploring_location and game.location != "安全屋":
                        button.draw()
                else:
                    button.draw()
    
    # 显示游戏消息
    message_bg = pygame.Rect(WIDTH//2 - 450, 450, 900, 50)
    pygame.draw.rect(screen, (0, 0, 0, 180), message_bg, border_radius=10)
    pygame.draw.rect(screen, (100, 100, 100), message_bg, 2, border_radius=10)
    
    message_text = font.render(game.message, True, YELLOW)
    screen.blit(message_text, (WIDTH//2 - message_text.get_width()//2, 465))
    
    # 显示安全屋加成提示
    if game.location == "安全屋" and not game.encountered_zombie and not game.show_combat_result:
        bonus_text = small_font.render("安全屋加成：资源使用效果+50%，休息效率翻倍", True, GREEN)
        screen.blit(bonus_text, (WIDTH//2 - bonus_text.get_width()//2, 500))
    
    # 显示探索状态提示
    if game.exploring_location and game.location != "安全屋" and not game.encountered_zombie and not game.show_combat_result:
        explore_info = small_font.render(f"正在探索{game.exploring_location}，可以继续探索或返回安全屋", True, GREEN)
        screen.blit(explore_info, (WIDTH//2 - explore_info.get_width()//2, 520))
    
    # 如果游戏结束
    if game.game_over:
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 200))
        screen.blit(overlay, (0, 0))
        
        # 结束游戏提示框
        end_bg = pygame.Rect(200, 250, 600, 250)
        pygame.draw.rect(screen, (50, 50, 50), end_bg, border_radius=20)
        pygame.draw.rect(screen, WHITE, end_bg, 3, border_radius=20)
        
        if game.survived:
            end_text = title_font.render("恭喜你成功生存了7天！", True, GREEN)
            sub_text = font.render("救援队伍已经到达，你安全了！", True, WHITE)
        else:
            end_text = title_font.render("游戏结束", True, RED)
            sub_text = font.render("在丧尸危机中生存并不容易...", True, WHITE)
        
        screen.blit(end_text, (WIDTH//2 - end_text.get_width()//2, HEIGHT//2 - 60))
        screen.blit(sub_text, (WIDTH//2 - sub_text.get_width()//2, HEIGHT//2 - 20))
        buttons["continue"].draw()
    
    pygame.display.flip()
    clock.tick(30)

pygame.quit()
sys.exit()
