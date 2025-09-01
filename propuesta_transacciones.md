# Análisis: Implementación de Múltiples Transacciones en PostRecharge y PostBuy

**Autores:**
- **Jean Zamora** - Desarrollador Backend (Si lo sueñas lo programo)

**Fecha de Creación:** Agosto 2025

---

## Resumen Ejecutivo

Este documento analiza la viabilidad y implicaciones de modificar las funciones `PostRecharge` y `PostBuy` para procesar múltiples transacciones en una sola petición HTTP, en lugar de procesar una transacción por petición como se hace actualmente.

## Estado Actual

### PostRecharge
- **Función**: Procesa una sola recarga de billetera
- **Campos actuales**: `amount`, `transactionDate`
- **Lógica**: Crea una transacción, actualiza saldo de billetera
- **Transacción DB**: Una sola transacción de base de datos

### PostBuy
- **Función**: Procesa una sola compra de boleto
- **Campos actuales**: `boletoId`, `idUsuarioApp`, `montoViaje`, `fechaHoraCompra`, `qrExpiracion`, `tipoBoleto`, `idEstacionUnidad`
- **Lógica**: Crea transacción, crea compra de boleto, actualiza saldo
- **Transacción DB**: Una sola transacción de base de datos

## Propuesta: Múltiples Transacciones

### 1. PostRecharge Múltiple

#### Estructura de Request Propuesta

**Request Individual (1 recarga)**:
```json
{
  "recharges": [
    {
      "amount": 100.00,
      "transactionDate": "1703123456"
    }
  ]
}
```

**Request Múltiple (3 recargas)**:
```json
{
  "recharges": [
    {
      "amount": 100.00,
      "transactionDate": "1703123456"
    },
    {
      "amount": 50.00,
      "transactionDate": "1703123457"
    },
    {
      "amount": 75.25,
      "transactionDate": "1703123458"
    }
  ]
}
```

**Endpoint**: Se mantiene el mismo: `POST /api/v1/wallets/recharge`

#### Estructura de Response Propuesta
```json
{
  "success": true,
  "message": "Recargas procesadas exitosamente",
  "data": {
    "old_balance": 0.00,
    "total_recharge": 225.25,
    "new_balance": 225.25,
    "transactions": [
      {
        "id": "uuid-1",
        "amount": 100.00,
        "transaction_date": "1703123457",
        "status": "completed"
      },
      {
        "id": "uuid-2",
        "amount": 50.00,
        "transaction_date": "1703123457",
        "status": "completed"
      },
      {
        "id": "uuid-3",
        "amount": 75.25,
        "transaction_date": "1703123457",
        "status": "completed"
      }
    ]
  }
}
```

### 2. PostBuy Múltiple

#### Estructura de Request Propuesta

**Request Individual (1 compra)**:
```json
{
  "purchases": [
    {
      "boletoId": "B001",
      "idUsuarioApp": "USER123",
      "montoViaje": 15.50,
      "fechaHoraCompra": "1703123456",
      "qrExpiracion": "1703123457",
      "tipoBoleto": "normal",
      "idEstacionUnidad": "EST001"
    }
  ]
}
```

**Request Múltiple (2 compras)**:
```json
{
  "purchases": [
    {
      "boletoId": "B001",
      "idUsuarioApp": "USER123",
      "montoViaje": 15.50,
      "fechaHoraCompra": "1703123456",
      "qrExpiracion": "1703123457",
      "tipoBoleto": "normal",
      "idEstacionUnidad": "EST001"
    },
    {
      "boletoId": "B002",
      "idUsuarioApp": "USER123",
      "montoViaje": 12.00,
      "fechaHoraCompra": "1703123457",
      "qrExpiracion": "1703123457",
      "tipoBoleto": "reducido",
      "idEstacionUnidad": "EST002"
    }
  ]
}
```

**Endpoint**: Se mantiene el mismo: `POST /api/v1/wallets/buy`

#### Estructura de Response Propuesta
```json
{
  "success": true,
  "message": "Compras realizadas exitosamente",
  "data": {
    "old_balance": 225.25,
    "total_purchase": 27.50,
    "new_balance": 197.75,
    "transactions": [
      {
        "id": "uuid-1",
        "boleto_info": {
          "boleto_id": "B001",
          "qr_expiracion": "1703123457",
          "tipo_boleto": "normal"
        },
        "amount": 15.50,
        "transaction_date": "1703123457",
        "status": "completed"
      },
      {
        "id": "uuid-2",
        "boleto_info": {
          "boleto_id": "B002",
          "qr_expiracion": "1703123457",
          "tipo_boleto": "reducido"
        },
        "amount": 12.00,
        "transaction_date": "1703123457",
        "status": "completed"
      }
    ]
  }
}
```

## Implicaciones Técnicas

### 1. Modificaciones en el Backend

#### Estructuras de Datos
- **Modificar estructuras existentes**: `RechargeRequest`, `BuyRequest` para soportar arrays
- **Validación**: Agregar validaciones para arrays de transacciones
- **Límites**: Definir límite máximo de transacciones por petición

#### Lógica de Negocio
- **Validación de saldo**: Para compras múltiples, verificar saldo total antes de procesar
- **Transacciones de DB**: Mantener una sola transacción de DB para atomicidad
- **Rollback**: Si falla una transacción, revertir todas las anteriores

#### Control de Errores
- **Validación individual**: Cada transacción debe validarse por separado
- **Respuestas parciales**: Opción de procesar transacciones válidas y reportar errores
- **Logging**: Mejorar logging para debugging de múltiples transacciones

### 2. Modificaciones en el Frontend

#### Manejo de Estados
- **Loading**: Indicador de progreso para múltiples transacciones
- **Validación**: Validar datos antes de enviar
- **Retry**: Mecanismo de reintento para transacciones fallidas

#### UX/UI
- **Formularios dinámicos**: Agregar/quitar transacciones
- **Validación en tiempo real**: Feedback inmediato sobre errores
- **Resumen**: Mostrar resumen antes de confirmar

## Ventajas de la Implementación

### 1. Rendimiento
- **Menos peticiones HTTP**: Reducción de overhead de red
- **Transacciones atómicas**: Garantía de consistencia de datos
- **Mejor throughput**: Procesamiento en lote más eficiente

### 2. Experiencia de Usuario
- **Operaciones en lote**: Usuarios pueden realizar múltiples operaciones de una vez
- **Menos interrupciones**: No hay necesidad de múltiples confirmaciones
- **Historial consolidado**: Transacciones relacionadas aparecen juntas

### 3. Negocio
- **Reducción de carga**: Menos peticiones al servidor
- **Mejor auditoría**: Transacciones relacionadas agrupadas
- **Flexibilidad**: Soporte para escenarios de uso complejos

## Desventajas y Riesgos

### 1. Complejidad
- **Lógica más compleja**: Manejo de múltiples transacciones
- **Validaciones múltiples**: Cada transacción debe validarse
- **Manejo de errores**: Casos edge más complejos

### 2. Rendimiento
- **Tiempo de respuesta**: Peticiones más largas para múltiples transacciones
- **Uso de memoria**: Mayor uso de memoria para procesar arrays
- **Bloqueo de DB**: Transacciones más largas pueden bloquear recursos

### 3. Mantenimiento
- **Código más complejo**: Más difícil de debuggear y mantener
- **Testing**: Más casos de prueba necesarios
- **Documentación**: API más compleja de documentar

## Ejemplos de Código a Modificar

### 1. Estructuras de Request (app/models/requests.go)

#### RechargeRequest Actual
```go
type RechargeRequest struct {
	Amount          float64 `json:"amount" binding:"required,gt=0"`
	TransactionDate string  `json:"transactionDate" binding:"required"`
}
```

#### RechargeRequest Modificado para Soporte Múltiple
```go
type RechargeRequest struct {
	Recharges []RechargeItem `json:"recharges" binding:"required,dive"`
}

type RechargeItem struct {
	Amount          float64 `json:"amount" binding:"required,gt=0"`
	TransactionDate string  `json:"transactionDate" binding:"required"`
}
```

#### BuyRequest Actual
```go
type BuyRequest struct {
	BoletoID         string  `json:"boletoId" binding:"required"`
	IDUsuarioApp     string  `json:"idUsuarioApp" binding:"required"`
	MontoViaje       float64 `json:"montoViaje" binding:"required,gt=0"`
	FechaHoraCompra  string  `json:"fechaHoraCompra" binding:"required"`
	QRExpiracion     string  `json:"qrExpiracion" binding:"required"`
	TipoBoleto       string  `json:"tipoBoleto" binding:"required"`
	IDEstacionUnidad string  `json:"idEstacionUnidad" binding:"required"`
}
```

#### BuyRequest Modificado para Soporte Múltiple
```go
type BuyRequest struct {
	Purchases []PurchaseItem `json:"purchases" binding:"required,dive"`
}

type PurchaseItem struct {
	BoletoID         string  `json:"boletoId" binding:"required"`
	IDUsuarioApp     string  `json:"idUsuarioApp" binding:"required"`
	MontoViaje       float64 `json:"montoViaje" binding:"required,gt=0"`
	FechaHoraCompra  string  `json:"fechaHoraCompra" binding:"required"`
	QRExpiracion     string  `json:"qrExpiracion" binding:"required"`
	TipoBoleto       string  `json:"tipoBoleto" binding:"required"`
	IDEstacionUnidad string  `json:"idEstacionUnidad" binding:"required"`
}
```

### 2. Controladores (app/controllers/wallet_controller.go)

#### PostRecharge - Lógica Simplificada
```go
func (wc *WalletController) PostRecharge(c *fiber.Ctx) error {
	var request models.RechargeRequest
	if err := c.BodyParser(&request); err != nil {
		return response.ErrorResponse(c, fiber.StatusBadRequest, "Error al procesar la solicitud")
	}

	// Validar que haya al menos una recarga
	if len(request.Recharges) == 0 {
		return response.ErrorResponse(c, fiber.StatusBadRequest, "Debe incluir al menos una recarga")
	}

	// Procesar todas las recargas (individual o múltiples)
	return wc.processRecharges(request.Recharges)
}

func (wc *WalletController) processRecharges(recharges []models.RechargeItem) error {
	// Lógica unificada para procesar recargas (1 o más)
	// ... implementación nueva ...
}
```

#### PostBuy - Lógica Simplificada
```go
func (wc *WalletController) PostBuy(c *fiber.Ctx) error {
	var request models.BuyRequest
	if err := c.BodyParser(&request); err != nil {
		return response.ErrorResponse(c, fiber.StatusBadRequest, "Error al procesar la solicitud")
	}

	// Validar que haya al menos una compra
	if len(request.Purchases) == 0 {
		return response.ErrorResponse(c, fiber.StatusBadRequest, "Debe incluir al menos una compra")
	}

	// Procesar todas las compras (individual o múltiples)
	return wc.processPurchases(request.Purchases)
}

func (wc *WalletController) processPurchases(purchases []models.PurchaseItem) error {
	// Lógica unificada para procesar compras (1 o más)
	// ... implementación nueva ...
}
```

### 3. Validaciones Adicionales
```go
// Validar límite de transacciones
const MaxTransactionsPerRequest = 20

func validateTransactionLimit(count int) error {
	if count > MaxTransactionsPerRequest {
		return fmt.Errorf("máximo %d transacciones por petición", MaxTransactionsPerRequest)
	}
	return nil
}

// Validar saldo total para compras múltiples
func validateTotalBalance(wallet models.Wallet, totalAmount float64) error {
	if !wallet.HasSufficientBalance(totalAmount) {
		return fmt.Errorf("saldo insuficiente. Saldo actual: $%.2f, monto total requerido: $%.2f", wallet.Balance, totalAmount)
	}
	return nil
}
```

## Consideraciones de Implementación

### 1. Límites y Validaciones
- **Máximo de transacciones**: Recomendado 10-20 por petición
- **Validación de saldo**: Para compras, verificar saldo total antes de procesar
- **Validación de datos**: Cada transacción debe cumplir con las reglas de negocio

### 2. Transacciones de Base de Datos
- **Atomicidad**: Mantener una sola transacción de DB
- **Rollback**: Revertir todas las operaciones si falla una
- **Performance**: Considerar índices y optimizaciones

### 3. Manejo de Errores
- **Validación temprana**: Validar todas las transacciones antes de procesar
- **Respuestas detalladas**: Informar qué transacciones fallaron y por qué
- **Logging**: Registrar información detallada para debugging

### 4. Compatibilidad
- **Endpoints existentes**: Se mantienen los mismos endpoints (`/recharge` y `/buy`)
- **Migración requerida**: El frontend existente debe migrar a la nueva estructura de arrays
- **Estructura unificada**: Tanto requests individuales como múltiples usan la misma estructura de array
- **Breaking changes**: Se requiere actualización del frontend para usar el nuevo formato

## Recomendaciones

### 1. Implementación Gradual
- **Fase 1**: Modificar endpoints existentes para usar estructura de arrays
- **Fase 2**: Migrar frontend a la nueva estructura de arrays
- **Fase 3**: Testing exhaustivo y validación en producción

### 2. Testing Exhaustivo
- **Casos de éxito**: Múltiples transacciones válidas
- **Casos de error**: Transacciones individuales inválidas
- **Casos mixtos**: Algunas válidas, algunas inválidas
- **Límites**: Máximo número de transacciones
- **Concurrencia**: Múltiples usuarios simultáneos

### 3. Monitoreo
- **Performance**: Tiempo de respuesta de nuevos endpoints
- **Errores**: Tasa de error y tipos de error
- **Uso**: Frecuencia de uso de endpoints múltiples vs individuales

## Conclusión

La implementación de múltiples transacciones en `PostRecharge` y `PostBuy` es técnicamente viable y ofrece beneficios significativos en términos de rendimiento y experiencia de usuario. La ventaja clave de este enfoque es que se mantienen los mismos endpoints existentes, simplificando la arquitectura del sistema.

La implementación se basa en una estructura unificada de arrays que puede contener desde una sola transacción hasta múltiples transacciones, eliminando la necesidad de lógica de detección compleja. Esto simplifica el código del backend y proporciona una API más consistente.

Sin embargo, requiere migración del frontend existente a la nueva estructura de arrays, lo que representa un breaking change. La implementación debe planificarse cuidadosamente con testing exhaustivo y una estrategia de migración clara para minimizar el impacto en los usuarios.
