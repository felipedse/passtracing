# Posicionamento de Imagem em WebXR

## Abordagem: Two-Handed Grab

O posicionamento da imagem e feito segurando ambos os grips simultaneamente. Isso permite mover, rotacionar (3 eixos) e escalar a imagem livremente no espaco 3D, sem etapa de calibracao.

### Como Funciona

1. **Inicio do grab**: Ao pressionar ambos os grips, calcula-se uma matriz de referencia ("grab space") a partir das posicoes e orientacoes dos dois controles, e armazena-se a transformacao relativa da imagem nesse espaco (`grabLocalMatrix`).

2. **Durante o grab**: A cada frame, recalcula-se a matriz de referencia com as novas posicoes/orientacoes e aplica-se a transformacao relativa armazenada.

3. **Fim do grab**: Ao soltar qualquer grip, a imagem permanece onde foi deixada.

### Calculo da Grab Matrix

A matriz e construida a partir de:

- **Posicao**: ponto medio entre os dois controles
- **Rotacao**: frame de coordenadas derivado de:
  - Eixo X: direcao do controle 1 ao controle 2
  - Eixo Y: "up" medio dos dois controles (via `getWorldQuaternion`)
  - Eixo Z: produto vetorial para completar o frame ortonormal
- **Escala**: tratada separadamente via razao de distancia entre controles

```javascript
function computeGrabMatrix(c1Pos, c2Pos, c1Quat, c2Quat) {
    const midpoint = c1Pos.clone().add(c2Pos).multiplyScalar(0.5);
    const direction = c2Pos.clone().sub(c1Pos).normalize();

    // Media do "up" real dos controles para rotacao 3-eixos
    const up1 = new THREE.Vector3(0, 1, 0).applyQuaternion(c1Quat);
    const up2 = new THREE.Vector3(0, 1, 0).applyQuaternion(c2Quat);
    const avgUp = up1.add(up2).normalize();

    let forward = new THREE.Vector3().crossVectors(avgUp, direction);
    forward.normalize();
    const correctedUp = new THREE.Vector3().crossVectors(direction, forward).normalize();

    const rotMatrix = new THREE.Matrix4().makeBasis(direction, correctedUp, forward);
    const quaternion = new THREE.Quaternion().setFromRotationMatrix(rotMatrix);

    return new THREE.Matrix4().compose(midpoint, quaternion, new THREE.Vector3(1, 1, 1));
}
```

### Graus de Liberdade

| Gesto | Efeito |
|-------|--------|
| Mover ambas as maos juntas | Translacao (XYZ) |
| Girar as maos como um volante | Rotacao no eixo entre controles |
| Inclinar ambas as maos frente/tras | Pitch |
| Inclinar ambas as maos lateralmente | Roll |
| Afastar/aproximar as maos | Escala (pinch-to-zoom) |

### Escala

A escala e calculada pela razao entre a distancia atual e a distancia inicial dos controles no momento do grab:

```javascript
currentScale = grabStartScale * (newDistance / grabStartDistance);
```

Limitada ao intervalo `[0.01, 4.0]`.

## Controles Adicionais

| Controle | Acao |
|----------|------|
| Analogico horizontal | Opacidade (+/-) |
| Analogico vertical | Escala (+/-) |
| A ou X | Mostrar/esconder imagem |
| B ou Y | Mostrar/esconder instrucoes |

Os controles de analogico sao desabilitados durante o grab para evitar conflitos.

## Armadilhas Comuns

### controller.position vs localToWorld

Em Three.js WebXR, o controller tem `matrixAutoUpdate = false` e a pose XR e aplicada direto na `matrix`/`matrixWorld`. Usar `.position` pode retornar (0,0,0) ou valor defasado. **Sempre usar:**

```javascript
// Posicao world correta
const worldPos = controller.localToWorld(new THREE.Vector3(0, 0, 0));

// Quaternion world correto
const worldQuat = new THREE.Quaternion();
controller.getWorldQuaternion(worldQuat);
```
