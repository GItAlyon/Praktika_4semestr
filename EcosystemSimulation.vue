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
        <input type="range" v-model="grassGrowthRate" min="5" max="10" step="0.5">
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
      <button @click="pauseSimulation">{{ isPaused ? 'Продолжить' : 'Пауза' }}</button>
      <button @click="resetSimulation">Сброс</button>
    </div>
    
    <div class="stats">
      <p>День: {{ day }}</p>
      <p>Травоядные: {{ currentHerbivores }}</p>
      <p>Хищники: {{ currentPredators }}</p>
      <p>Трава: {{ Math.round(currentGrass) }}</p>
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

      <div class="grass-container">
        <div class="grass-growth">
          <div class="grass-amount" 
            :style="{ width: grassPercentage + '%' }">
          </div>
          <span class="grass-label">
            Трава: {{ Math.round(currentGrass) }}/{{ maxGrass }}
            (рост: {{ formattedGrassRate }} ед./день)
          </span>
        </div>
      </div>

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

      //<div class="grass-growth">
      //  <div class="grass-bar" :style="{ width: grassPercentage + '%' }"></div>
      //  <span class="grass-text">Трава: {{ Math.round(currentGrass) }}/{{ maxGrass }}</span>
      //</div>
      
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
    this.speed = 1 + Math.random() * 2;
    this.reproductionCooldown = 0;
  }

  move(width, height) {
    // Периодически меняем направление
    if (Math.random() < 0.05) {
      this.direction = Math.random() * Math.PI * 2;
    }
    
    this.x += Math.cos(this.direction) * this.speed;
    this.y += Math.sin(this.direction) * this.speed;
    
    // Ограничиваем движение в пределах экосистемы
    this.x = Math.max(0, Math.min(width, this.x));
    this.y = Math.max(0, Math.min(height, this.y));
    
    // Тратим энергию на движение
    this.energy -= 0.1;
   
    if (this.reproductionCooldown > 0) {
      this.reproductionCooldown--;
    }
  }

  eat(grass) {
    if (grass > 0) {
      const eaten = Math.min(10, grass);
      this.energy += eaten * 0.5;
      return eaten;
    }
    return 0;
  }

  canReproduce() {
    return this.energy > 80 && this.reproductionCooldown === 0;
  }

  reproduce() {
    this.energy /= 2;
    this.reproductionCooldown = 50;
    return new Herbivore(this.x + (Math.random() * 20 - 10), this.y + (Math.random() * 20 - 10));
  }
}

class Predator {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.energy = 100;
    this.direction = Math.random() * Math.PI * 2;
    this.speed = 2 + Math.random() * 3;
    this.reproductionCooldown = 0;
    this.huntCooldown = 0;
  }

  move(width, height) {
    // Периодически меняем направление
    if (Math.random() < 0.05) {
      this.direction = Math.random() * Math.PI * 2;
    }
    
    this.x += Math.cos(this.direction) * this.speed;
    this.y += Math.sin(this.direction) * this.speed;
    
    // Ограничиваем движение в пределах экосистемы
    this.x = Math.max(0, Math.min(width, this.x));
    this.y = Math.max(0, Math.min(height, this.y));
    
    // Тратим энергию на движение
    this.energy -= 0.2;
    
    if (this.reproductionCooldown > 0) {
      this.reproductionCooldown--;
    }
    if (this.huntCooldown > 0) {
      this.huntCooldown--;
    }
  }

  canHunt(huntFrequency) {
    return this.huntCooldown === 0 && Math.random() < huntFrequency * 0.01;
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
    return this.energy > 120 && this.reproductionCooldown === 0;
  }

  reproduce() {
    this.energy /= 2;
    this.reproductionCooldown = 100;
    return new Predator(this.x + (Math.random() * 20 - 10), this.y + (Math.random() * 20 - 10));
  }
}

export default {
  data() {
    return {
      // Параметры
      herbivoreCount: 5,
      predatorCount: 3,
      grassPatches: [], // Массив участков травы
      grassPatchSize: 20, // Размер участка травы
      maxGrassPatches: 50, // Максимальное количество участков
      grassGrowthRate: 7, // Скорость роста травы 
      minGrassGrowth: 5,  // Минимальный рост
      maxGrassGrowth: 10,  // Максимальный рост
      huntFrequency: 3,
      simulationSpeed: 1, // Переменная для контроля скорости (1x по умолчанию)
      minSpeed: 0.5,      // Минимальная скорость (0.5x)
      maxSpeed: 5,        // Максимальная скорость (5x)
      baseInterval: 500,  // Базовый интервал (500ms для 1x скорости)
      
      // Состояние симуляции
      herbivores: [],
      predators: [],
      currentGrass: 100, // Начальное количество травы 100
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
      if (!this.isPaused && this.simulationInterval) {
        clearInterval(this.simulationInterval);
        const interval = this.baseInterval / this.simulationSpeed;
        this.simulationInterval = setInterval(() => {
          this.simulateDay();
        }, interval);
      }
    },
    
    pauseSimulation() {
      this.isPaused = !this.isPaused;
    },
    
    resetSimulation() {
      // Очищаем предыдущую симуляцию
      clearInterval(this.simulationInterval);
      
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
      this.currentGrass = 100;
      this.day = 0;
      this.isPaused = true;

      // Инициализируем участки травы
      this.grassPatches = [];
      this.generateGrassPatches(10); // Начальное количество участков
    },

    generateGrassPatches(count) {
      for (let i = 0; i < count; i++) {
        this.addGrassPatch();
      }
    },

    addGrassPatch() {
      if (this.grassPatches.length < this.maxGrassPatches) {
        this.grassPatches.push({
          x: Math.random() * this.ecosystemWidth,
          y: Math.random() * this.ecosystemHeight,
          amount: 5 + Math.random() * 5, // Количество травы на участке
          size: 15 + Math.random() * 10 // Размер участка
        });
      }
    },
    
    simulateDay() {
      this.day++;
      
      // Рост травы - добавляем новые участки
      if (this.day % 3 === 0 && this.grassPatches.length < this.maxGrassPatches) {
        this.addGrassPatch();
      }
      
      // Обновляем количество травы
      this.currentGrass = this.grassPatches.reduce((sum, patch) => sum + patch.amount, 0);

      //if (this.currentGrass < this.maxGrass) {
      //  this.currentGrass = Math.min(
      //    this.maxGrass,
      //    this.currentGrass + Number(this.grassGrowthRate)
      // );
      //}

      const grassPerHerbivore = 0.5; // Необходимо 0.5 ед. травы на одно травоядное
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
      let totalEaten = 0;
      const newHerbivores = [];
      
      for (const herbivore of this.herbivores) {
        if (herbivore.energy > 0) {
          if (this.day > 30 && Math.random() < 0.05) {
            herbivore.energy = 0;
            continue;
          }
          herbivore.move(this.ecosystemWidth, this.ecosystemHeight);
          
          // Травоядные едят не более 0.5 единицы травы в день
          const eaten = Math.min(0.5, this.currentGrass - totalEaten);
          if (eaten > 0) {
            totalEaten += herbivore.eat(eaten);
          }
          
          // Размножение раз в 5 дней при достаточной энергии
          if (this.day % 5 === 0 && herbivore.canReproduce()) {
            newHerbivores.push(herbivore.reproduce());
          }
        }
      }
      
      this.currentGrass = Math.max(0, this.currentGrass - totalEaten);
      
      // Добавляем новых травоядных
      this.herbivores = [...this.herbivores.filter(h => h.energy > 0), ...newHerbivores];
      
      // Движение и охота хищников
      const newPredators = [];
      
      for (const predator of this.predators) {
        if (predator.energy > 0) {
          if (predator.energy > 0 && this.day > 30 && Math.random() < 0.03) {
          predator.energy = 0;
          } 
          predator.move(this.ecosystemWidth, this.ecosystemHeight);
          if (this.day % 6 === 0 && predator.canReproduce()) {
            newPredators.push(predator.reproduce());
          }
          //if (predator.canHunt(this.huntFrequency)) {
          //  predator.hunt(this.herbivores);
          //}
          
          //if (predator.canReproduce()) {
         //   newPredators.push(predator.reproduce());
          //}
        }
      }
      
      // Добавляем новых хищников
      this.predators = [...this.predators.filter(p => p.energy > 0), ...newPredators];
      
      // Удаляем мертвых особей
      this.herbivores = this.herbivores.filter(h => h.energy > 0);
      this.predators = this.predators.filter(p => p.energy > 0);
      
      // Если вымерли все травоядные или хищники, перезапускаем симуляцию
      if (this.currentHerbivores === 0 || this.currentPredators === 0) {
        setTimeout(() => {
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
