<template>
  <div class="ecosystem-container">
    <h1>Симуляция экосистемы</h1>
    
    <div class="controls">
      <div class="control-group">
        <label>Количество травоядных: {{ herbivoreCount }}</label>
        <input type="range" v-model="herbivoreCount" min="1" max="20">
      </div>
      
      <div class="control-group">
        <label>Количество хищников: {{ predatorCount }}</label>
        <input type="range" v-model="predatorCount" min="1" max="20">
      </div>
      
      <div class="control-group">
        <label>Скорость роста травы: {{ grassGrowthRate }}</label>
        <input type="range" v-model="grassGrowthRate" min="1" max="7" step="0.5">
      </div>
      
      <div class="control-group">
        <label>Частота охоты хищников: {{ huntFrequency }}</label>
        <input type="range" v-model="huntFrequency" min="1" max="10">
      </div>

      <div class="control-group">
        <label>Скорость симуляции: {{ simulationSpeed }}x</label>
        <input 
          type="range" 
          v-model="simulationSpeed" 
          :min="minSpeed" 
          :max="maxSpeed" 
          step="0.5"
          @change="updateSimulationSpeed"
        >
      </div>
      
      <button @click="startSimulation">Старт</button>
      <button @click="pauseSimulation" :disabled="!simulationRunning">{{ isPaused ? 'Продолжить' : 'Пауза' }}</button>
      <button @click="resetSimulation">Сброс</button>
    </div>
    
    <div class="stats">
      <p>День: {{ day }}</p>
      <p>Травоядные: {{ currentHerbivores }}</p>
      <p>Хищники: {{ currentPredators }}</p>
      <p>Трава: {{ Math.round(currentGrass) }}/{{ maxGrass }}</p>
      <p class="color-info">Обозначения цветов: 
        <span style="color: #3498db;">синим</span> - травоядные, 
        <span style="color: #e74c3c;">красным</span> - хищники, 
        <span style="color: #4CAF50;">зелёным</span> - трава
      </p>
    </div>
    
    <div class="ecosystem" ref="ecosystem">
      <!-- Участки травы -->
      <div 
        v-for="(patch, index) in grassPatches" 
        :key="'grass'+index"
        class="grass-patch"
        :style="{
        left: patch.x + 'px',
        top: patch.y + 'px',
        width: patch.size + 'px',
        height: patch.size + 'px',
        opacity: patch.amount / 10
        }"
      ></div>

      <div 
        v-for="(herbivore, index) in herbivores" 
        :key="'herbivore'+index"
        class="entity herbivore"
        :style="{
          left: herbivore.x + 'px',
          top: herbivore.y + 'px',
          opacity: herbivore.energy / 100
        }"
      ></div>
      
      <div 
        v-for="(predator, index) in predators" 
        :key="'predator'+index"
        class="entity predator"
        :style="{
          left: predator.x + 'px',
          top: predator.y + 'px',
          opacity: predator.energy / 100
        }"
      ></div>

      <div 
        v-for="(herbivore, index) in herbivores" 
        :key="'herbivore'+index"
        class="entity herbivore"
        :class="{ dying: herbivore.energy <= 0 }"
        :style="{
          left: herbivore.x + 'px',
          top: herbivore.y + 'px',
          opacity: herbivore.energy / 100
        }"
      ></div>
      
      <div 
        class="grass-indicator"
        :style="{
          backgroundColor: `rgba(0, 200, 0, ${currentGrass / maxGrass})`
        }"
      ></div>
    </div>
  </div>
</template>

<script>
class Herbivore {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.energy = 100;
    this.direction = Math.random() * Math.PI * 2;
    this.baseSpeed = 1.5 + Math.random(); // Базовая скорость
    this.hungerSpeedMultiplier = 1.5; // +50% скорости при голоде
    this.currentSpeed = this.baseSpeed;
    this.reproductionCooldown = 0;
    this.hungerThreshold = 40; // Порог энергии для активации "режима голода"
  }

  updateSpeed() {
    // Ускоряемся, если энергия <= порога
    this.currentSpeed = this.energy <= this.hungerThreshold 
      ? this.baseSpeed * this.hungerSpeedMultiplier 
      : this.baseSpeed;
  }

  move(width, height, grassPatches) {
    this.updateSpeed(); // Обновляем скорость перед движением

    // Поиск травы при голоде
    if (this.energy <= this.hungerThreshold) {
      const nearestGrass = this.findNearestGrass(grassPatches);
      if (nearestGrass) {
        this.direction = Math.atan2(
          nearestGrass.y - this.y, 
          nearestGrass.x - this.x
        );
      }
    } 
    // Случайное направление в обычном режиме
    else if (Math.random() < 0.05) {
      this.direction = Math.random() * Math.PI * 2;
    }

    // Движение с текущей скоростью
    this.x += Math.cos(this.direction) * this.currentSpeed;
    this.y += Math.sin(this.direction) * this.currentSpeed;

    // Границы экосистемы
    this.x = Math.max(0, Math.min(width, this.x));
    this.y = Math.max(0, Math.min(height, this.y));

    // Затраты энергии (больше при ускорении)
    this.energy -= this.currentSpeed > this.baseSpeed ? 0.13 : 0.1;

    if (this.reproductionCooldown > 0) {
      this.reproductionCooldown--;
    }
  }

  eat(grassPatches) {
    const eatingRadius = 20; // Радиус, в котором травоядное может есть траву
  
    // Ищем ближайший участок травы
    for (const patch of grassPatches) {
      if (!patch.hasGrass()) continue; // Пропускаем пустые участки
    
      const distance = Math.sqrt(
        Math.pow(this.x - patch.x, 2) + 
        Math.pow(this.y - patch.y, 2)
      );
    
      // Если участок травы в радиусе поедания
      if (distance < eatingRadius) {
        const canEat = Math.min(2, patch.amount); // Может съесть до 2 единиц за раз
        patch.loseAmount(canEat);
        this.energy += canEat * 0.5;
      
        // Если съели достаточно, прекращаем поиск
        if (canEat >= 2) break;
      }
    }
  }

  findNearestGrass(grassPatches) {
    let nearestPatch = null;
    let minDistance = Infinity;

    for (const patch of grassPatches) {
      if (patch.amount <= 0) continue;
      
      const distance = Math.sqrt(
        Math.pow(this.x - patch.x, 2) + 
        Math.pow(this.y - patch.y, 2)
      );
      
      if (distance < minDistance) {
        minDistance = distance;
        nearestPatch = patch;
      }
    }
    return nearestPatch;
  }

  canReproduce() {
    return this.energy > 80 && this.reproductionCooldown === 0;
  }

  reproduce() {
    const childEnergy = this.energy * 0.5; // Дитё получает 50% энергии родителя
    this.energy *= 0.6; // Родитель сохраняет 50%
    this.reproductionCooldown = 50;

    const child = new Herbivore(
      this.x + (Math.random() * 20 - 10),
      this.y + (Math.random() * 20 - 10)
    );

    child.energy = childEnergy; // Явно задаём энергию потомку
    return child;
  }
}

class Predator {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.energy = 100;
    this.direction = Math.random() * Math.PI * 2;
    this.baseSpeed = 2 + Math.random(); // Базовая скорость
    this.huntSpeed = this.baseSpeed * 2.5; // Скорость в режиме охоты
    this.currentSpeed = this.baseSpeed;
    this.target = null; // Цель для охоты
    this.reproductionCooldown = 0;
    this.huntCooldown = 0;
  }

  move(width, height) {
    // Если есть цель и она еще жива
    if (this.target && this.target.energy > 0) {
      const distance = Math.sqrt(
        Math.pow(this.x - this.target.x, 2) + 
        Math.pow(this.y - this.target.y, 2)
      );
      
      // Если цель в радиусе 50px - переходим в режим погони
      if (distance < 50) {
        this.currentSpeed = this.huntSpeed;
        
        // Двигаемся прямо к цели
        this.direction = Math.atan2(
          this.target.y - this.y, 
          this.target.x - this.x
        );
      } else {
        // Цель слишком далеко - сбрасываем цель
        this.target = null;
        this.currentSpeed = this.baseSpeed;
      }
    } else {
      // Режим случайного блуждания
      this.currentSpeed = this.baseSpeed;
      if (Math.random() < 0.05) {
        this.direction = Math.random() * Math.PI * 2;
      }
    }
    
    // Движение с текущей скоростью
    this.x += Math.cos(this.direction) * this.currentSpeed;
    this.y += Math.sin(this.direction) * this.currentSpeed;
    
    // Ограничиваем движение в пределах экосистемы
    this.x = Math.max(0, Math.min(width, this.x));
    this.y = Math.max(0, Math.min(height, this.y));
    
    // Тратим энергию на движение (больше в режиме охоты)
    this.energy -= this.currentSpeed === this.huntSpeed ? 0.3 : 0.15;
    
    // Уменьшаем кулдауны
    if (this.reproductionCooldown > 0) this.reproductionCooldown--;
    if (this.huntCooldown > 0) this.huntCooldown--;
  }

  canHunt(huntFrequency) {
    return this.huntCooldown === 0 && Math.random() < huntFrequency * 0.03;
  }

  hunt(herbivores) {
    let closest = null;
    let minDist = Infinity;
    
    for (const herbivore of herbivores) {
      const dist = Math.sqrt(
        Math.pow(this.x - herbivore.x, 2) + 
        Math.pow(this.y - herbivore.y, 2)
      );
      
      if (dist < 50 && dist < minDist && herbivore.energy > 0) {
        closest = herbivore;
        minDist = dist;
      }
    }
    
    if (closest) {
      closest.energy = 0; // "Съедаем" травоядного
      this.energy += 30;
      this.huntCooldown = 10;
      return true;
    }
    return false;
  }

  canReproduce() {
    return this.energy > 99 && this.reproductionCooldown === 0;
  }

  reproduce() {
    const childEnergy = this.energy * 0.5;
    this.energy *= 0.6;
    this.reproductionCooldown = 70;

    const child = new Predator(
      this.x + (Math.random() * 20 - 10),
      this.y + (Math.random() * 20 - 10)
    );
    child.energy = childEnergy;
    return child;
  }
}

class Grass {
  constructor(x, y, amount, size) {
    this.x = x;
    this.y = y;
    this.amount = amount;
    this.size = size;
    this.maxAmount = 10; // Максимальное количество травы на участке
  }

  // Рост травы
  grow(growthRate) {
    this.amount = Math.min(this.maxAmount, this.amount + growthRate * 0.3);
    return this.amount;
  }

  // Потеря травы при поедании
  loseAmount(value) {
    this.amount = Math.max(0, this.amount - value);
    return this.amount;
  }

  // Проверка, есть ли трава на участке
  hasGrass() {
    return this.amount > 0;
  }
}

export default {
  data() {
    return {
      // Параметры
      herbivoreCount: 7,
      predatorCount: 7,
      grassPatches: [], // Массив участков травы
      grassPatchSize: 20, // Размер участка травы
      maxGrassPatches: 30, // Максимальное количество участков
      grassGrowthRate: 3, // Скорость роста травы по умолчанию
      minGrassGrowth: 1,  // Минимальный рост
      maxGrassGrowth: 7,  // Максимальный рост
      huntFrequency: 5,
      simulationSpeed: 1, // Переменная для контроля скорости по умолчанию
      minSpeed: 0.5,      // Минимальная скорость (0.5x)
      maxSpeed: 5,        // Максимальная скорость (5x)
      baseInterval: 900,  // Базовый интервал (900ms для 1x скорости)
      simulationEnded: false, // Флаг завершения симуляции
      alertTimeout: null,     // Таймер для alert
      simulationRunning: false, // Флаг, указывающий, что симуляция запущена
      
      // Состояние симуляции
      herbivores: [],
      predators: [],
      currentGrass: 0, // Начальное количество травы 0
      maxGrass: 200,
      day: 0,
      isPaused: false,
      simulationInterval: null,
      ecosystemWidth: 0,
      ecosystemHeight: 0
    };
  },
  
  computed: {
    formattedGrassRate() {
    return Number(this.grassGrowthRate).toFixed(1);
    },
    grassPercentage() {
      return (this.currentGrass / this.maxGrass) * 100;
    },
    currentHerbivores() {
      return this.herbivores.filter(h => h.energy > 0).length;
    },
    currentPredators() {
      return this.predators.filter(p => p.energy > 0).length;
    }
  },
  
  mounted() {
    this.resetSimulation();
    this.updateEcosystemSize();
    window.addEventListener('resize', this.updateEcosystemSize);
  },
  
  beforeUnmount() {
    clearInterval(this.simulationInterval);
    window.removeEventListener('resize', this.updateEcosystemSize);
  },
  
  methods: {
    updateEcosystemSize() {
      if (this.$refs.ecosystem) {
        this.ecosystemWidth = this.$refs.ecosystem.clientWidth;
        this.ecosystemHeight = this.$refs.ecosystem.clientHeight;
      }
    },
    
    startSimulation() {
      this.resetSimulation();
      this.isPaused = false;
      this.simulationRunning = true; // Симуляция запущена
      
      if (this.simulationInterval) {
        clearInterval(this.simulationInterval);
      }
      
      // Учитываем скорость симуляции
      const interval = this.baseInterval / this.simulationSpeed;
      this.simulationInterval = setInterval(() => {
        if (!this.isPaused) {
          this.simulateDay();
        }
      }, interval);
    },

    updateSimulationSpeed() {
      if (this.simulationInterval) {
        const wasPaused = this.isPaused; // Сохраняем состояние паузы
        clearInterval(this.simulationInterval);
        const interval = this.baseInterval / this.simulationSpeed;
    
        this.simulationInterval = setInterval(() => {
          if (!this.isPaused) {
            this.simulateDay();
          }
        }, interval);
    
        this.isPaused = wasPaused; // Восстанавливаем состояние паузы
      }
    },
    
    pauseSimulation() {
      if (this.simulationRunning) // Только если симуляция запущена
        this.isPaused = !this.isPaused;
    },
    
    resetSimulation() {
      // Очищаем предыдущую симуляцию
      clearInterval(this.simulationInterval);
      clearTimeout(this.alertTimeout); // Очищаем таймер
      this.simulationEnded = false;    // Сбрасываем флаг
      this.simulationRunning = false; // Симуляция остановлена
      this.isPaused = true;
      
      // Создаем новых травоядных
      this.herbivores = [];
      for (let i = 0; i < this.herbivoreCount; i++) {
        this.herbivores.push(new Herbivore(
          Math.random() * this.ecosystemWidth,
          Math.random() * this.ecosystemHeight
        ));
      }
      
      // Создаем новых хищников
      this.predators = [];
      for (let i = 0; i < this.predatorCount; i++) {
        this.predators.push(new Predator(
          Math.random() * this.ecosystemWidth,
          Math.random() * this.ecosystemHeight
        ));
      }
      
      // Сбрасываем траву и день
      this.currentGrass = 0;
      this.day = 0;
      this.isPaused = true;

      // Инициализируем участки травы
      this.grassPatches = [];
      this.generateGrassPatches(6); // Начальное количество участков
    },

    generateGrassPatches(count) {
      for (let i = 0; i < count; i++) {
        this.addGrassPatch();
      }
    },

    addGrassPatch() {
      if (this.grassPatches.length < this.maxGrassPatches) {
        const totalGrass = this.grassPatches.reduce((sum, patch) => sum + patch.amount, 0);
        const availableSpace = Math.max(0, this.maxGrass - totalGrass);
  
        if (availableSpace > 0) {
          const newAmount = Math.min(5 + Math.random() * 5, availableSpace);
          if (newAmount > 0) {
            this.grassPatches.push(new Grass(
              Math.random() * this.ecosystemWidth,
              Math.random() * this.ecosystemHeight,
              newAmount,
              15 + Math.random() * 10
            ));
         }
        }
      }
    },
    
    simulateDay() {
      if (this.simulationEnded) return; // Прекращаем если симуляция уже завершена
      this.day++;
      
      // Рост травы - добавляем новые участки
      if (this.day % 3 === 0 && this.grassPatches.length < this.maxGrassPatches && this.currentGrass < this.maxGrass) {
        // Увеличиваем количество травы на всех участках
        this.grassPatches.forEach(patch => {
          patch.grow(this.grassGrowthRate);
        });

        // Иногда добавляем новые участки
        if (Math.random() < 0.3 && this.grassPatches.length < this.maxGrassPatches) {
          this.addGrassPatch();
        }
      }
      
      // Обновляем количество травы
      this.currentGrass = Math.min(
        this.maxGrass,
        this.grassPatches.reduce((sum, patch) => sum + patch.amount, 0)
      );

      const grassPerHerbivore = 2; // Необходимо 2 ед. травы на одно травоядное
      if (this.currentGrass < this.currentHerbivores * grassPerHerbivore) {
        const deficit = (this.currentHerbivores * grassPerHerbivore) - this.currentGrass;
        const herbivoresToDie = Math.floor(deficit / grassPerHerbivore);
        
        // Убиваем случайных травоядных
        const aliveHerbivores = this.herbivores.filter(h => h.energy > 0);
        for (let i = 0; i < Math.min(herbivoresToDie, aliveHerbivores.length); i++) {
          const randomIndex = Math.floor(Math.random() * aliveHerbivores.length);
          aliveHerbivores[randomIndex].energy = 0;
        }
      }
      
      // Движение и питание травоядных
      const newHerbivores = [];
      
      for (const herbivore of this.herbivores) {
        if (herbivore.energy > 0) {
          if (this.day > 30 && Math.random() < 0.04) {
            herbivore.energy = 0;
            continue;
          }
          herbivore.move(this.ecosystemWidth, this.ecosystemHeight, this.grassPatches);
          
          // Травоядные едят траву только если находятся рядом с ней
          herbivore.eat(this.grassPatches);
          
          // Размножение раз в 5 дней при достаточной энергии
          if (this.day % 5 === 0 && herbivore.canReproduce()) {
            newHerbivores.push(herbivore.reproduce());
          }
        }
      }
      
      // Обновляем общее количество травы после поедания
      this.currentGrass = Math.max(0, 
       this.grassPatches.reduce((sum, patch) => sum + patch.amount, 0)
      );      

      // Добавляем новых травоядных
      this.herbivores = [...this.herbivores.filter(h => h.energy > 0), ...newHerbivores];
      
      // Движение и охота хищников
      const newPredators = [];
      
      for (const predator of this.predators) {
        if (predator.energy > 0) {
          // Смерть от старости
          if (this.day > 30 && Math.random() < 0.04) {
          predator.energy = 0;
          continue;
          } 

          // Выбираем цель для охоты если нет текущей
          if (!predator.target && predator.canHunt(this.huntFrequency)) {
            const aliveHerbivores = this.herbivores.filter(h => h.energy > 0);
            if (aliveHerbivores.length > 0) {
              // Находим ближайшего травоядного
              let closest = null;
              let minDist = Infinity;
              
              for (const herbivore of aliveHerbivores) {
                const dist = Math.sqrt(
                  Math.pow(predator.x - herbivore.x, 2) + 
                  Math.pow(predator.y - herbivore.y, 2)
                );
                
                if (dist < minDist) {
                  minDist = dist;
                  closest = herbivore;
                }
              }
              
              if (closest && minDist < 100) { // Макс дистанция для начала охоты
                predator.target = closest;
              }
            }
          }

          // Движение (передаем массив травоядных для поиска целей)
          predator.move(this.ecosystemWidth, this.ecosystemHeight, this.herbivores);
          
          // Проверка атаки если есть цель
          if (predator.target && predator.target.energy > 0) {
            const distToTarget = Math.sqrt(
              Math.pow(predator.x - predator.target.x, 2) + 
              Math.pow(predator.y - predator.target.y, 2)
            );
            
            // Если достаточно близко - атакуем
            if (distToTarget < 15) { // Радиус атаки
              predator.target.energy = 0;
              predator.energy += 30;
              predator.huntCooldown = 15; // Долгий кулдаун после удачной охоты
              predator.target = null; // Сбрасываем цель
            }
          }
          // Размножение
          if (this.day % 10 === 0 && predator.canReproduce()) {
            newPredators.push(predator.reproduce());
          }
          
        }
      }
      
      // Добавляем новых хищников
      this.predators = [...this.predators.filter(p => p.energy > 0), ...newPredators];
      
      // Удаляем мертвых особей
      this.herbivores = this.herbivores.filter(h => h.energy > 0);
      this.predators = this.predators.filter(p => p.energy > 0);
      
      // Если вымерли все травоядные или хищники, перезапускаем симуляцию
      if ((this.currentHerbivores === 0 || this.currentPredators === 0) && !this.simulationEnded) {
        this.simulationEnded = true;
        this.simulationRunning = false; // Симуляция завершена
        clearTimeout(this.alertTimeout); // Очищаем предыдущий таймер
        this.alertTimeout = setTimeout(() => {
          alert(`Симуляция завершена на день ${this.day}. ${this.currentHerbivores === 0 ? 'Вымерли травоядные' : 'Вымерли хищники'}`);
          this.resetSimulation();
        }, 100);
      }
    }
  }
};
</script>

<style scoped>
.ecosystem-container {
  max-width: 800px;
  margin: 0 auto;
  font-family: Arial, sans-serif;
}

.controls {
  background: #f0f0f0;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.control-group {
  margin-bottom: 10px;
}

.control-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.control-group input {
  width: 100%;
}

button {
  margin-right: 10px;
  padding: 5px 10px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background: #45a049;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

button:disabled:hover {
  background: #cccccc;
}

.stats {
  background: #e9f7ef;
  padding: 10px;
  border-radius: 8px;
  margin-bottom: 15px;
}

.stats p {
  margin: 5px 0;
  font-weight: bold;
}

.color-info {
  font-size: 0.9em;
  color: #555;
  margin-top: 8px;
}

.ecosystem {
  position: relative;
  width: 100%;
  height: 500px;
  background-color: #e8f4f8;
  border: 2px solid #333;
  border-radius: 8px;
  overflow: hidden;
}

.entity {
  position: absolute;
  width: 15px;
  height: 15px;
  border-radius: 50%;
  transform: translate(-50%, -50%);
}

.herbivore {
  background-color: #3498db;
}

.predator {
  background-color: #e74c3c;
}

.grass-patch {
  position: absolute;
  background-color: #4CAF50;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  transition: all 0.3s ease;
  border: 1px solid #2E8B57;
}
</style>
