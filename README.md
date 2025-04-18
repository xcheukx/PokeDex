# Vue 3 + TypeScript + Vite

This template should help get you started developing with Vue 3 and TypeScript in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about the recommended Project Setup and IDE Support in the [Vue Docs TypeScript Guide](https://vuejs.org/guide/typescript/overview.html#project-setup).

node version 18.6.0

## pokeapi相关

以下是关于 **Pokémon API** 的详细指南和实现方案：

---

### 一、官方 Pokémon API

**PokeAPI** (<https://pokeapi.co>) 是最常用的免费 Pokémon 数据接口，支持中英文数据。

#### 基础使用示例（JavaScript）

```javascript
// 获取皮卡丘的完整数据
fetch('https://pokeapi.co/api/v2/pokemon/pikachu')
  .then(response => response.json())
  .then(data => {
    console.log('名称:', data.name);        // 英文名
    console.log('身高:', data.height);      // 分米单位
    console.log('体重:', data.weight);      // 百克单位
    console.log('技能:', data.moves.map(m => m.move.name)); 
  });

// 获取中文名称（需要二次请求）
fetch('https://pokeapi.co/api/v2/pokemon-species/25') // 25是皮卡丘的ID
  .then(res => res.json())
  .then(data => {
    const zhName = data.names.find(n => n.language.name === 'zh-Hans');
    console.log('中文名:', zhName?.name || '未知');
  });
```

---

### 二、核心接口清单

| 接口类型          | 示例URL                              | 返回内容                  |
|-------------------|--------------------------------------|--------------------------|
| 宝可梦基础信息    | `/api/v2/pokemon/{id或name}`        | 身高/体重/技能/图片URL   |
| 宝可梦种类数据    | `/api/v2/pokemon-species/{id}`      | 进化链/中文名/图鉴描述   |
| 属性相克表        | `/api/v2/type/{type-name}`          | 克制关系（2x/0.5x伤害）  |
| 游戏版本数据      | `/api/v2/version/{version-name}`    | 红宝石/蓝宝石等版本信息  |

---

### 三、中文数据优化方案

#### 1. 直接获取中文结果

```javascript
// 获取妙蛙种子的中文信息（需使用species接口）
fetch('https://pokeapi.co/api/v2/pokemon-species/1')
  .then(res => res.json())
  .then(data => {
    // 中文名称
    const zhName = data.names.find(n => n.language.name === 'zh-Hans');
    
    // 中文图鉴描述（过滤非中文版本）
    const zhEntry = data.flavor_text_entries.find(
      e => e.language.name === 'zh-Hans'
    );
    
    console.log(`${zhName.name}：${zhEntry.flavor_text}`);
  });
```

#### 2. 批量处理技巧

```javascript
// 并行请求前10个宝可梦的中英文数据
const requests = Array.from({ length: 10 }, (_, i) => 
  Promise.all([
    fetch(`https://pokeapi.co/api/v2/pokemon/${i + 1}`),
    fetch(`https://pokeapi.co/api/v2/pokemon-species/${i + 1}`)
  ]).then(([baseRes, speciesRes]) => 
    Promise.all([baseRes.json(), speciesRes.json()])
  )
);

Promise.all(requests).then(results => {
  results.forEach(([baseData, speciesData]) => {
    const zhName = speciesData.names.find(n => n.language.name === 'zh-Hans');
    console.log(`No.${baseData.id} ${zhName.name}`);
  });
});
```

---

### 四、进阶使用场景

#### 1. 属性相克计算器

```javascript
// 获取火属性相克关系
fetch('https://pokeapi.co/api/v2/type/fire')
  .then(res => res.json())
  .then(data => {
    const doubleDamageTo = data.damage_relations.double_damage_to;
    console.log('火属性克制:',
      doubleDamageTo.map(t => t.name).join(', '));
  });
```

#### 2. 进化链可视化

```javascript
// 获取伊布的进化链（需要递归处理）
async function getEvolutionChain(id) {
  const speciesRes = await fetch(`https://pokeapi.co/api/v2/pokemon-species/${id}`);
  const { evolution_chain } = await speciesRes.json();
  
  const chainRes = await fetch(evolution_chain.url);
  const chainData = await chainRes.json();
  
  // 递归解析进化链
  function parseChain(chain) {
    const pokemon = chain.species.name;
    const evolvesTo = chain.evolves_to.map(parseChain);
    return { pokemon, evolvesTo };
  }
  
  return parseChain(chainData.chain);
}

// 使用示例
getEvolutionChain(133).then(chain => {
  console.log(JSON.stringify(chain, null, 2));
});
```

---

### 五、注意事项

1. **速率限制**：每分钟最多100次请求（建议添加延迟）

   ```javascript
   // 请求间添加300ms延迟
   async function throttledFetch(url) {
     await new Promise(resolve => setTimeout(resolve, 300));
     return fetch(url);
   }
   ```

2. **数据缓存**：推荐使用本地缓存（如localStorage）：

   ```javascript
   async function getCachedPokemon(id) {
     const cacheKey = `pokemon-${id}`;
     const cached = localStorage.getItem(cacheKey);
     if (cached) return JSON.parse(cached);
     
     const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
     const data = await res.json();
     localStorage.setItem(cacheKey, JSON.stringify(data));
     return data;
   }
   ```

3. **HTTPS要求**：部分浏览器会阻止混合内容请求，确保页面使用HTTPS

如需实现具体功能（如队伍战斗力计算器、属性相克表等），可进一步说明需求方向。
