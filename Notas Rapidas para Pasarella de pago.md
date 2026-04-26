- Dto para crear pagos (Esto es lo que envia el front)

```
amount: number;

currency: string;

description: string;

token?: string;

payment_method_id?: string;

issuer_id?: string;

installments?: number;

metadata?: Record<string, any>;
```

Ejemplo de payload para crear qr:

{
  "externalPosId": "CAJA_001",
  "externalReference": "PEDIDO-12345",
  "title": "Compra en Punto de Venta",
  "description": "Venta de productos varios",
  "totalAmount": 1500.50,
  "items": [
    {
      "skuNumber": "PROD-01",
      "category": "electronics",
      "title": "Auriculares Bluetooth",
      "unitPrice": 1000.00,
      "quantity": 1,
      "unitMeasure": "unit",
      "totalAmount": 1000.00
    },
    {
      "skuNumber": "PROD-02",
      "category": "accessories",
      "title": "Funda para celular",
      "unitPrice": 250.25,
      "quantity": 2,
      "unitMeasure": "unit",
      "totalAmount": 500.50
    }
  ]
}
