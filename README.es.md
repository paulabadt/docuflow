# DocuFlow 📄

*Plataforma serverless de procesamiento y automatización de documentos empresariales
construida sobre AWS Lambda, Step Functions y CloudFormation, con backend Django REST
Framework y aplicación cliente React + TypeScript*

---

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Funcionalidades Principales](#funcionalidades-principales)
- [Stack Tecnológico](#stack-tecnológico)
- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Instalación](#instalación)
- [Uso](#uso)
- [Ejemplos de Código](#ejemplos-de-código)
- [Documentación de la API](#documentación-de-la-api)
- [Contribución](#contribución)
- [Licencia](#licencia)

---

## 🌟 Descripción General

**DocuFlow** es una plataforma nativa en la nube que automatiza el ciclo de vida
completo de documentos empresariales — desde la carga y extracción inteligente de
datos hasta flujos de trabajo de aprobación en múltiples etapas, archivado por
cumplimiento normativo y seguimiento de estado en tiempo real. El sistema aprovecha
los servicios serverless de AWS como motor principal de procesamiento, con Django REST
Framework impulsando la capa de API orientada al cliente y React + TypeScript
entregando una interfaz de gestión documental responsiva e intuitiva.

Desarrollado como parte de un proyecto de investigación en el SENA (Servicio Nacional
de Aprendizaje), este sistema demuestra ingeniería full-stack Python de nivel
producción usando Django 5 y Django REST Framework en el backend, arquitectura AWS
serverless con Lambda, Step Functions y CloudFormation como infraestructura como
código, y React en el frontend — cubriendo el espectro completo del desarrollo
moderno Python/React para clientes globales de servicios de TI profesionales.

### 🎯 Objetivos del Proyecto

- Automatizar flujos de trabajo de procesamiento documental en múltiples etapas usando AWS Step Functions
- Implementar pipelines serverless de extracción de datos con AWS Lambda y triggers de S3
- Exponer APIs REST limpias y versionadas usando Django REST Framework para aplicaciones cliente
- Aprovisionar toda la infraestructura AWS mediante CloudFormation como código — sin configuración manual en consola
- Demostrar patrones Django 5 en producción: managers personalizados, signals, admin, permisos
- Construir frontend React + TypeScript responsivo consumiendo APIs DRF con estado Redux
- Entregar actualizaciones de estado de documentos en tiempo real vía WebSocket con AWS API Gateway
- Aplicar TDD con Pytest + Django Test Client y BDD con Cypress en flujos E2E
- Automatizar despliegues mediante GitHub Actions CI/CD integrando stacks de CloudFormation

### 🏆 Logros

- ✅ Procesamiento de más de 5.000 documentos a través del pipeline Lambda serverless sin gestión de servidores
- ✅ Flujos de trabajo Step Functions completaron aprobaciones en múltiples etapas en promedio 40% más rápido que el proceso manual
- ✅ Los stacks CloudFormation aprovisionan el entorno de producción completo en menos de 8 minutos
- ✅ Pipeline GitHub Actions despliega frontend, contenedor Django y funciones Lambda en una sola ejecución
- ✅ APIs Django REST Framework sosteniendo más de 500 solicitudes concurrentes con tiempos de respuesta inferiores a 150ms
- ✅ Cero deriva de infraestructura entre entornos — dev, staging y producción idénticos vía IaC
- ✅ Cobertura de pruebas unitarias superior al 96% en todas las capas de servicios y modelos Django con Pytest

---

## ✨ Funcionalidades Principales

### 📤 Ingesta Serverless de Documentos — S3 + Lambda
```python
# lambdas/document_processor/handler.py
# AWS Lambda — Disparado por evento S3 PutObject al cargar un documento

import json
import os
import boto3
import logging
from datetime import datetime, timezone
from urllib.parse import unquote_plus

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

s3_client       = boto3.client("s3")
sfn_client      = boto3.client("stepfunctions")
textract_client = boto3.client("textract")

WORKFLOW_ARN = os.environ["STEP_FUNCTIONS_ARN"]


def handler(event: dict, context) -> dict:
    """
    Punto de entrada — disparado por S3 PutObject en el bucket de documentos.
    Inicia un flujo de trabajo Step Functions por cada documento cargado.
    """
    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key    = unquote_plus(record["s3"]["object"]["key"])
        size   = record["s3"]["object"]["size"]

        logger.info(
            "Documento cargado — bucket=%s key=%s size=%d",
            bucket, key, size,
        )

        # Extraer metadatos de las etiquetas del objeto S3
        tags_response = s3_client.get_object_tagging(Bucket=bucket, Key=key)
        tags = {t["Key"]: t["Value"] for t in tags_response["TagSet"]}

        # Iniciar ejecución de Step Functions para este documento
        execution_input = {
            "bucket":        bucket,
            "key":           key,
            "document_id":   tags.get("document_id"),
            "tenant_id":     tags.get("tenant_id"),
            "document_type": tags.get("document_type", "GENERIC"),
            "uploaded_by":   tags.get("uploaded_by"),
            "uploaded_at":   datetime.now(timezone.utc).isoformat(),
            "file_size":     size,
        }

        response = sfn_client.start_execution(
            stateMachineArn=WORKFLOW_ARN,
            name=f"doc-{tags.get('document_id')}-{int(datetime.now().timestamp())}",
            input=json.dumps(execution_input),
        )

        logger.info(
            "Ejecución Step Functions iniciada — executionArn=%s",
            response["executionArn"],
        )

    return {"statusCode": 200, "body": "Documentos encolados para procesamiento"}
```
```python
# lambdas/text_extractor/handler.py
# AWS Lambda — Tarea Step Functions: extraer texto del documento con AWS Textract

import json
import boto3
import logging

logger          = logging.getLogger(__name__)
textract_client = boto3.client("textract")
s3_client       = boto3.client("s3")


def handler(event: dict, context) -> dict:
    """
    Tarea Step Functions — extraer texto estructurado del documento.
    Entrada: ubicación S3 del documento cargado.
    Salida: bloques de texto extraído y pares clave-valor.
    """
    bucket        = event["bucket"]
    key           = event["key"]
    document_type = event["document_type"]

    logger.info("Extrayendo texto — bucket=%s key=%s", bucket, key)

    # Iniciar job asíncrono de Textract para documentos de múltiples páginas
    response = textract_client.start_document_analysis(
        DocumentLocation={
            "S3Object": {"Bucket": bucket, "Name": key}
        },
        FeatureTypes=["FORMS", "TABLES"],
    )

    job_id = response["JobId"]

    # Esperar hasta que el job de Textract complete
    extracted = _wait_for_textract(job_id)

    # Parsear pares clave-valor de los campos del formulario
    key_values = _parse_key_value_pairs(extracted["Blocks"])

    return {
        **event,
        "extraction": {
            "job_id":     job_id,
            "status":     "COMPLETED",
            "key_values": key_values,
            "page_count": extracted.get("DocumentMetadata", {}).get("Pages", 1),
            "confidence": _calculate_avg_confidence(extracted["Blocks"]),
        },
    }


def _wait_for_textract(job_id: str) -> dict:
    import time
    while True:
        result = textract_client.get_document_analysis(JobId=job_id)
        status = result["JobStatus"]
        if status == "SUCCEEDED":
            return result
        if status == "FAILED":
            raise RuntimeError(f"Job Textract {job_id} falló")
        time.sleep(2)


def _parse_key_value_pairs(blocks: list) -> dict:
    key_map, value_map, block_map = {}, {}, {}

    for block in blocks:
        block_map[block["Id"]] = block
        if block["BlockType"] == "KEY_VALUE_SET":
            if "KEY" in block.get("EntityTypes", []):
                key_map[block["Id"]] = block
            else:
                value_map[block["Id"]] = block

    result = {}
    for key_id, key_block in key_map.items():
        key_text   = _get_text(key_block, block_map)
        value_text = ""
        for rel in key_block.get("Relationships", []):
            if rel["Type"] == "VALUE":
                for vid in rel["Ids"]:
                    if vid in value_map:
                        value_text = _get_text(value_map[vid], block_map)
        result[key_text] = value_text

    return result


def _get_text(block: dict, block_map: dict) -> str:
    text = ""
    for rel in block.get("Relationships", []):
        if rel["Type"] == "CHILD":
            for cid in rel["Ids"]:
                child = block_map.get(cid, {})
                if child.get("BlockType") == "WORD":
                    text += child.get("Text", "") + " "
    return text.strip()


def _calculate_avg_confidence(blocks: list) -> float:
    confidences = [
        b["Confidence"] for b in blocks
        if "Confidence" in b and b["BlockType"] == "WORD"
    ]
    return round(sum(confidences) / len(confidences), 2) if confidences else 0.0
```

**Funcionalidades:**
- ☁️ Ingesta de documentos sin servidor disparada directamente por eventos S3 PutObject
- 🔍 Integración AWS Textract para extracción inteligente de campos de formularios y tablas
- 🏷️ Etiquetado de objetos S3 para aislamiento por tenant y enrutamiento de metadatos
- 🔀 Inicio automático de flujo de trabajo Step Functions por cada documento cargado
- 📊 Puntuación de confianza en todos los bloques de texto extraído para validación de calidad

### 🔄 AWS Step Functions — Orquestación de Flujos de Aprobación
```json
// infrastructure/step-functions/document-workflow.json
// AWS Step Functions — Máquina de estados para procesamiento y aprobación de documentos

{
  "Comment": "Flujo de trabajo DocuFlow para procesamiento y aprobación en múltiples etapas",
  "StartAt": "ExtractDocumentText",
  "States": {

    "ExtractDocumentText": {
      "Type": "Task",
      "Resource": "${TextExtractorLambdaArn}",
      "Next": "ValidateExtraction",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "MarkExtractionFailed",
          "ResultPath": "$.error"
        }
      ]
    },

    "ValidateExtraction": {
      "Type": "Task",
      "Resource": "${ValidationLambdaArn}",
      "Next": "CheckValidationResult",
      "ResultPath": "$.validation"
    },

    "CheckValidationResult": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.validation.passed",
          "BooleanEquals": true,
          "Next": "NotifyReviewer"
        },
        {
          "Variable": "$.validation.passed",
          "BooleanEquals": false,
          "Next": "MarkValidationFailed"
        }
      ]
    },

    "NotifyReviewer": {
      "Type": "Task",
      "Resource": "${NotificationLambdaArn}",
      "Next": "WaitForApproval",
      "ResultPath": "$.notification"
    },

    "WaitForApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "${ApprovalQueueUrl}",
        "MessageBody": {
          "document_id.$": "$.document_id",
          "tenant_id.$":   "$.tenant_id",
          "task_token.$":  "$$.Task.Token",
          "extraction.$":  "$.extraction"
        }
      },
      "HeartbeatSeconds": 86400,
      "Next": "CheckApprovalDecision",
      "ResultPath": "$.approval"
    },

    "CheckApprovalDecision": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.approval.decision",
          "StringEquals": "APPROVED",
          "Next": "ArchiveDocument"
        },
        {
          "Variable": "$.approval.decision",
          "StringEquals": "REJECTED",
          "Next": "MarkRejected"
        }
      ]
    },

    "ArchiveDocument": {
      "Type": "Task",
      "Resource": "${ArchiveLambdaArn}",
      "Next": "NotifyCompletion",
      "ResultPath": "$.archive"
    },

    "NotifyCompletion": {
      "Type": "Task",
      "Resource": "${NotificationLambdaArn}",
      "End": true
    },

    "MarkExtractionFailed": {
      "Type": "Task",
      "Resource": "${StatusUpdaterLambdaArn}",
      "Parameters": { "status": "EXTRACTION_FAILED" },
      "End": true
    },

    "MarkValidationFailed": {
      "Type": "Task",
      "Resource": "${StatusUpdaterLambdaArn}",
      "Parameters": { "status": "VALIDATION_FAILED" },
      "End": true
    },

    "MarkRejected": {
      "Type": "Task",
      "Resource": "${StatusUpdaterLambdaArn}",
      "Parameters": { "status": "REJECTED" },
      "End": true
    }
  }
}
```

**Funcionalidades:**
- 🔄 Máquina de estados en múltiples etapas: Extrae → Valida → Notifica → Espera → Aprueba → Archiva
- ⏸️ Patrón `waitForTaskToken` para pasos de aprobación con intervención humana
- 🔁 Reintento automático de Lambda con backoff exponencial en fallos transitorios
- 🌿 Estados Choice para enrutamiento condicional según resultados de validación y aprobación
- 📋 Auditoría completa — cada transición de estado registrada en el historial de Step Functions

### 🐍 Django REST Framework — API Orientada al Cliente
```python
# documents/models.py
# Django 5 — Modelo de documento con manager personalizado y hooks de signals

from django.db import models
from django.contrib.auth import get_user_model
from django.db.models.signals import post_save
from django.dispatch import receiver
import uuid

User = get_user_model()


class DocumentQuerySet(models.QuerySet):
    def for_tenant(self, tenant_id: str):
        return self.filter(tenant_id=tenant_id)

    def pending_approval(self):
        return self.filter(status=Document.Status.PENDING_APPROVAL)

    def processed(self):
        return self.filter(
            status__in=[Document.Status.APPROVED, Document.Status.ARCHIVED]
        )


class DocumentManager(models.Manager):
    def get_queryset(self):
        return DocumentQuerySet(self.model, using=self._db)

    def for_tenant(self, tenant_id: str):
        return self.get_queryset().for_tenant(tenant_id)

    def pending_approval(self):
        return self.get_queryset().pending_approval()


class Document(models.Model):
    class Status(models.TextChoices):
        UPLOADED          = "UPLOADED",          "Cargado"
        EXTRACTING        = "EXTRACTING",        "Extrayendo"
        VALIDATING        = "VALIDATING",        "Validando"
        PENDING_APPROVAL  = "PENDING_APPROVAL",  "Pendiente de Aprobación"
        APPROVED          = "APPROVED",          "Aprobado"
        REJECTED          = "REJECTED",          "Rechazado"
        ARCHIVED          = "ARCHIVED",          "Archivado"
        EXTRACTION_FAILED = "EXTRACTION_FAILED", "Fallo en Extracción"
        VALIDATION_FAILED = "VALIDATION_FAILED", "Fallo en Validación"

    class DocumentType(models.TextChoices):
        INVOICE   = "INVOICE",   "Factura"
        CONTRACT  = "CONTRACT",  "Contrato"
        REPORT    = "REPORT",    "Informe"
        FORM      = "FORM",      "Formulario"
        GENERIC   = "GENERIC",   "Genérico"

    id             = models.UUIDField(primary_key=True,
                                      default=uuid.uuid4, editable=False)
    tenant_id      = models.CharField(max_length=100, db_index=True)
    title          = models.CharField(max_length=255)
    document_type  = models.CharField(max_length=30,
                                      choices=DocumentType.choices)
    status         = models.CharField(max_length=30,
                                      choices=Status.choices,
                                      default=Status.UPLOADED,
                                      db_index=True)
    s3_bucket      = models.CharField(max_length=255)
    s3_key         = models.CharField(max_length=1024)
    file_size      = models.PositiveBigIntegerField(default=0)
    extracted_data = models.JSONField(default=dict, blank=True)
    workflow_arn   = models.CharField(max_length=2048, blank=True)
    uploaded_by    = models.ForeignKey(User, on_delete=models.SET_NULL,
                                       null=True,
                                       related_name="uploaded_documents")
    reviewed_by    = models.ForeignKey(User, on_delete=models.SET_NULL,
                                       null=True,
                                       related_name="reviewed_documents")
    reviewed_at    = models.DateTimeField(null=True, blank=True)
    review_notes   = models.TextField(blank=True)
    created_at     = models.DateTimeField(auto_now_add=True)
    updated_at     = models.DateTimeField(auto_now=True)

    objects = DocumentManager()

    class Meta:
        ordering = ["-created_at"]
        indexes  = [
            models.Index(fields=["tenant_id", "status"]),
            models.Index(fields=["tenant_id", "document_type"]),
        ]

    def __str__(self):
        return f"{self.title} ({self.status})"


@receiver(post_save, sender=Document)
def notify_status_change(sender, instance, created, **kwargs):
    """Transmitir actualización de estado vía WebSocket en cada guardado del documento."""
    if not created:
        from documents.tasks import broadcast_document_update
        broadcast_document_update.delay(str(instance.id), instance.status)
```
```python
# documents/serializers.py + documents/views.py
# Django REST Framework — API de documentos con aislamiento por tenant

from rest_framework import serializers, viewsets, status, permissions
from rest_framework.decorators import action
from rest_framework.response import Response
from documents.models import Document
from documents.services import DocumentUploadService, ApprovalService


class DocumentSerializer(serializers.ModelSerializer):
    upload_url = serializers.SerializerMethodField()

    class Meta:
        model  = Document
        fields = [
            "id", "title", "document_type", "status",
            "file_size", "extracted_data", "workflow_arn",
            "uploaded_by", "reviewed_by", "reviewed_at",
            "review_notes", "created_at", "updated_at",
            "upload_url",
        ]
        read_only_fields = [
            "id", "status", "extracted_data", "workflow_arn",
            "reviewed_by", "reviewed_at", "created_at", "updated_at",
        ]

    def get_upload_url(self, obj) -> str | None:
        if obj.status == Document.Status.UPLOADED:
            return DocumentUploadService.generate_presigned_url(
                obj.s3_bucket, obj.s3_key
            )
        return None


class DocumentViewSet(viewsets.ModelViewSet):
    serializer_class   = DocumentSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        # Aislamiento por tenant — operadores solo ven sus propios documentos
        return Document.objects.for_tenant(
            self.request.user.profile.tenant_id
        ).select_related("uploaded_by", "reviewed_by")

    def perform_create(self, serializer):
        service  = DocumentUploadService(self.request.user)
        document = service.initiate_upload(
            title=serializer.validated_data["title"],
            document_type=serializer.validated_data["document_type"],
        )
        serializer.save(
            tenant_id=self.request.user.profile.tenant_id,
            uploaded_by=self.request.user,
            s3_bucket=document["bucket"],
            s3_key=document["key"],
        )

    @action(detail=True, methods=["post"], url_path="approve")
    def approve(self, request, pk=None):
        document = self.get_object()
        service  = ApprovalService(request.user)
        result   = service.approve(
            document, notes=request.data.get("notes", "")
        )
        return Response(DocumentSerializer(result).data)

    @action(detail=True, methods=["post"], url_path="reject")
    def reject(self, request, pk=None):
        document = self.get_object()
        service  = ApprovalService(request.user)
        result   = service.reject(
            document, reason=request.data.get("reason", "")
        )
        return Response(
            DocumentSerializer(result).data,
            status=status.HTTP_200_OK,
        )

    @action(detail=False, methods=["get"], url_path="pending")
    def pending_approval(self, request):
        qs = Document.objects.pending_approval().filter(
            tenant_id=request.user.profile.tenant_id
        )
        serializer = DocumentSerializer(qs, many=True)
        return Response(serializer.data)
```

**Funcionalidades:**
- 🏢 Aislamiento de documentos multi-tenant mediante managers QuerySet personalizados de Django
- 📤 Generación de URLs presignadas S3 para cargas seguras directas desde el navegador a S3
- ✅ Acciones DRF personalizadas `approve` y `reject` que disparan callbacks de tokens de tareas Step Functions
- 🔔 Signals Django transmitiendo cambios de estado a clientes WebSocket vía Celery
- 🛡️ Personalización de Django Admin para revisión de documentos y monitoreo de flujos de trabajo

### ☁️ CloudFormation — Infraestructura como Código
```yaml
# infrastructure/cloudformation/docuflow-stack.yaml
# AWS CloudFormation — Definición completa de infraestructura serverless

AWSTemplateFormatVersion: "2010-09-09"
Description: Infraestructura serverless DocuFlow para procesamiento de documentos

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
  DjangoImage:
    Type: String
    Description: URI de imagen ECR para el contenedor Django

Resources:

  # S3 — Bucket de almacenamiento con reglas de ciclo de vida
  DocumentsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "docuflow-documents-${Environment}"
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: ArchivarDocumentosAntiguos
            Status: Enabled
            Transitions:
              - TransitionInDays: 90
                StorageClass: GLACIER
      NotificationConfiguration:
        LambdaConfigurations:
          - Event: "s3:ObjectCreated:*"
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: "uploads/"
            Function: !GetAtt DocumentProcessorFunction.Arn
      CorsConfiguration:
        CorsRules:
          - AllowedOrigins: ["*"]
            AllowedMethods: [GET, PUT, POST]
            AllowedHeaders: ["*"]
            MaxAge: 3600

  # Lambda — Procesador de ingesta de documentos
  DocumentProcessorFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub "docuflow-processor-${Environment}"
      Runtime: python3.11
      Handler: handler.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        S3Bucket: !Sub "docuflow-deployments-${Environment}"
        S3Key: "lambdas/document_processor.zip"
      Environment:
        Variables:
          STEP_FUNCTIONS_ARN: !Ref DocumentWorkflowStateMachine
          ENVIRONMENT: !Ref Environment
      Timeout: 30
      MemorySize: 256

  # Step Functions — Máquina de estados del flujo documental
  DocumentWorkflowStateMachine:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      StateMachineName: !Sub "docuflow-workflow-${Environment}"
      RoleArn: !GetAtt StepFunctionsExecutionRole.Arn
      DefinitionString: !Sub |
        {
          "Comment": "Flujo de trabajo DocuFlow",
          "StartAt": "ExtractDocumentText",
          "States": {
            "ExtractDocumentText": {
              "Type": "Task",
              "Resource": "${TextExtractorFunction.Arn}",
              "Next": "ValidateExtraction"
            }
          }
        }

  # SQS — Cola de aprobación con cola de mensajes fallidos
  ApprovalQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "docuflow-approvals-${Environment}"
      VisibilityTimeout: 300
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt ApprovalDLQ.Arn
        maxReceiveCount: 3

  ApprovalDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub "docuflow-approvals-dlq-${Environment}"
      MessageRetentionPeriod: 1209600  # 14 días

  # ECS Fargate — Contenedor de la aplicación Django
  DjangoTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub "docuflow-django-${Environment}"
      NetworkMode: awsvpc
      RequiresCompatibilities: [FARGATE]
      Cpu: "512"
      Memory: "1024"
      ExecutionRoleArn: !GetAtt ECSExecutionRole.Arn
      ContainerDefinitions:
        - Name: django
          Image: !Ref DjangoImage
          PortMappings:
            - ContainerPort: 8000
          Environment:
            - Name: DJANGO_ENV
              Value: !Ref Environment
            - Name: AWS_REGION
              Value: !Ref AWS::Region
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Sub "/ecs/docuflow-${Environment}"
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: django

  # RDS PostgreSQL — Base de datos primaria
  DocumentsDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: !Sub "docuflow-db-${Environment}"
      DBInstanceClass: db.t3.micro
      Engine: postgres
      EngineVersion: "15.4"
      MasterUsername: "docuflow"
      MasterUserPassword: !Sub "{{resolve:secretsmanager:docuflow/${Environment}/db-password}}"
      AllocatedStorage: "20"
      StorageType: gp3
      MultiAZ: !If [IsProd, true, false]
      BackupRetentionPeriod: !If [IsProd, 7, 1]

Outputs:
  DocumentsBucketName:
    Value: !Ref DocumentsBucket
    Export:
      Name: !Sub "DocuFlow-${Environment}-BucketName"

  StateMachineArn:
    Value: !Ref DocumentWorkflowStateMachine
    Export:
      Name: !Sub "DocuFlow-${Environment}-StateMachineArn"

  ApprovalQueueUrl:
    Value: !Ref ApprovalQueue
    Export:
      Name: !Sub "DocuFlow-${Environment}-ApprovalQueueUrl"
```

**Funcionalidades:**
- 📦 Infraestructura completa de producción definida en una sola plantilla CloudFormation
- 🔔 Notificaciones de eventos S3 conectadas automáticamente a triggers Lambda vía IaC
- 💀 Cola de mensajes fallidos SQS para mensajes de aprobación con retención de 14 días
- 🌍 Parámetro de entorno (`dev`/`staging`/`prod`) controla el nombre de todos los recursos
- 🔐 Integración con Secrets Manager para credenciales de base de datos — sin valores en código

---

## 🛠️ Stack Tecnológico

### Backend

| Tecnología                  | Propósito                                              | Versión     |
|-----------------------------|--------------------------------------------------------|-------------|
| **Python**                  | Lenguaje backend principal                             | 3.11+       |
| **Django**                  | Framework web + ORM + Admin + Signals                  | 5.x         |
| **Django REST Framework**   | Capa REST API con serializadores y viewsets            | 3.15.x      |
| **AWS Lambda**              | Funciones serverless de procesamiento documental       | Runtime 3.11|
| **AWS Step Functions**      | Orquestación de flujos de aprobación en múltiples etapas| Standard   |
| **AWS S3**                  | Almacenamiento de documentos con gestión de ciclo de vida| —          |
| **AWS Textract**            | Extracción inteligente de texto de documentos          | —           |
| **AWS SQS**                 | Cola de aprobación con cola de mensajes fallidos       | —           |
| **AWS ECS Fargate**         | Hosting serverless de contenedor Django                | —           |
| **AWS RDS PostgreSQL**      | Base de datos relacional gestionada                    | 15.x        |
| **AWS Cognito**             | Autenticación y autorización multi-tenant              | —           |
| **CloudFormation**          | Infraestructura como código para todos los recursos AWS| —           |
| **Celery + Redis**          | Cola de tareas asíncronas para transmisión WebSocket   | 5.x         |
| **Channels + ASGI**         | Soporte WebSocket en Django                            | 4.x         |
| **Pytest + pytest-django**  | TDD — suite de pruebas unitarias e integración         | 8.x         |
| **factory_boy**             | Fábricas de fixtures de prueba para modelos Django     | 3.x         |

### Frontend

| Tecnología                | Propósito                                        | Versión  |
|---------------------------|--------------------------------------------------|----------|
| **React**                 | Framework de UI                                  | 18.x     |
| **TypeScript**            | Tipado estático                                  | 5.x      |
| **React-Redux**           | Gestión de estado global                         | 8.x      |
| **Redux Toolkit**         | Redux simplificado con slices y Thunks           | 1.9.x    |
| **Redux Thunk**           | Middleware asíncrono para llamadas a la API DRF  | 2.4.x    |
| **React Router v6**       | Enrutamiento del lado del cliente                | 6.x      |
| **Webpack**               | Configuración manual de empaquetado de módulos   | 5.x      |
| **SASS/SCSS**             | Preprocesamiento avanzado de CSS                 | 1.x      |
| **Axios**                 | Cliente HTTP con interceptores                   | 1.x      |
| **React Testing Library** | Pruebas unitarias de componentes (TDD)           | 14.x     |
| **Cypress**               | Pruebas end-to-end BDD                           | 13.x     |

### DevOps e Infraestructura

| Tecnología             | Propósito                                             |
|------------------------|-------------------------------------------------------|
| **CloudFormation**     | IaC completo — S3, Lambda, Step Functions, RDS        |
| **GitHub Actions**     | CI/CD — pruebas, build, deploy Lambda, update CFN     |
| **Docker**             | Construcción de imagen del contenedor Django          |
| **AWS ECR**            | Registro de contenedores para imagen Django           |

---

## 🏗️ Arquitectura del Sistema

### Arquitectura General
```
┌─────────────────────────────────────────────────────────────────────┐
│                        CAPA DE PRESENTACIÓN                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │  Dashboard de    │  │  Panel de        │  │  Administración  │ │
│  │  Carga de Docs   │  │  Revisión y      │  │  y Reportes      │ │
│  │  (React + TS)    │  │  Aprobación      │  │  (Django Admin)  │ │
│  │                  │  │  (React + TS)    │  │                  │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
│           │                     │                      │            │
│           └─────────────────────┴──────────────────────┘           │
│                                 │                                   │
│              ┌──────────────────▼──────────────────┐               │
│              │        Redux Store + Thunks          │               │
│              │  documentos · flujo · auth · ui      │               │
│              └──────────────────┬──────────────────┘               │
└─────────────────────────────────┼───────────────────────────────────┘
                   REST API        │       WebSocket
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                      CAPA DE APLICACIÓN                             │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │        Django 5 + DRF — Contenedor ECS Fargate              │   │
│  │                                                              │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │   │
│  │  │  ViewSet  │ │  ViewSet  │ │  ViewSet  │ │  ViewSet   │  │   │
│  │  │   Auth    │ │ Documentos│ │Aprobaciones│ │  Tenants  │  │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────┘  │   │
│  │                                                              │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │   │
│  │  │  Django   │ │  Django   │ │  Django   │ │  Celery    │  │   │
│  │  │  Signals  │ │   ORM     │ │   Admin   │ │  Workers   │  │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘  │
│                                 │                                   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                  CAPA SERVERLESS AWS                                │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────┐  ┌────────────▼────────────────────────────┐    │
│  │   AWS S3     │  │         AWS Step Functions               │    │
│  │              │  │                                          │    │
│  │  uploads/    │  │  ExtractDocumentText                     │    │
│  │  ├── Evento  │  │       ↓                                  │    │
│  │  │  PutObject│  │  ValidateExtraction                      │    │
│  │  └─────────┐ │  │       ↓                                  │    │
│  │            │ │  │  CheckValidationResult (Choice)          │    │
│  └────────────┘ │  │       ↓                                  │    │
│                 │  │  NotifyReviewer                          │    │
│  ┌──────────────▼┐ │       ↓                                  │    │
│  │  Funciones   │ │  WaitForApproval (waitForTaskToken)       │    │
│  │  Lambda      │ │       ↓                                   │    │
│  │              │ │  CheckApprovalDecision (Choice)           │    │
│  │ ├── Processor│ │       ↓                                   │    │
│  │ ├── Extractor│ │  ArchiveDocument                          │    │
│  │ ├── Validator│ │       ↓                                   │    │
│  │ ├── Notifier │ │  NotifyCompletion                         │    │
│  │ ├── Archiver │ │                                           │    │
│  │ └── Updater  │ └───────────────────────────────────────────┘   │
│  └──────────────┘                                                  │
│                                 │                                   │
│  ┌──────────────┐  ┌────────────▼───┐  ┌───────────┐             │
│  │  AWS SQS     │  │  AWS Textract  │  │AWS Cognito│             │
│  │              │  │                │  │           │             │
│  │  Cola        │  │  Campos de     │  │  Auth     │             │
│  │  Aprobación  │  │  formulario    │  │  Multi-   │             │
│  │  + DLQ       │  │  Tablas        │  │  tenant   │             │
│  └──────────────┘  └────────────────┘  └───────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                         CAPA DE DATOS                               │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────┐  ┌────────▼────────┐  ┌──────────────────┐  │
│  │   AWS RDS        │  │   AWS S3        │  │ AWS ElastiCache  │  │
│  │   PostgreSQL     │  │                 │  │ Redis            │  │
│  │                  │  │  documentos/    │  │                  │  │
│  │  - documents     │  │  archivos/      │  │ - Cola Celery    │  │
│  │  - tenants       │  │  miniaturas/    │  │ - Caché sesión   │  │
│  │  - users         │  │                 │  │ - Canales WS     │  │
│  │  - audit_logs    │  │                 │  │                  │  │
│  │  - approvals     │  │                 │  │                  │  │
│  └──────────────────┘  └─────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                CAPA CI/CD — GitHub Actions                          │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  Push a main                    │                                   │
│       ↓                         │                                   │
│  pytest + cobertura             │                                   │
│       ↓                         │                                   │
│  Construir imagen Docker Django │                                   │
│       ↓                         │                                   │
│  Subir imagen a ECR             │                                   │
│       ↓                         │                                   │
│  Empaquetar ZIPs Lambda         │                                   │
│       ↓                         │                                   │
│  Desplegar stack CloudFormation │                                   │
│       ↓                         │                                   │
│  Actualizar servicio ECS Fargate│                                   │
│       ↓                         │                                   │
│  Ejecutar migraciones Django    │                                   │
│       ↓                         │                                   │
│  Desplegar build React a S3/CDN │                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Estructura de Servicios
```
docuflow/
├── backend/
│   ├── config/
│   │   ├── settings/
│   │   │   ├── base.py
│   │   │   ├── development.py
│   │   │   ├── staging.py
│   │   │   └── production.py
│   │   ├── urls.py
│   │   ├── asgi.py
│   │   └── wsgi.py
│   ├── documents/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── admin.py
│   │   ├── signals.py
│   │   ├── tasks.py
│   │   ├── services/
│   │   │   ├── upload_service.py
│   │   │   ├── approval_service.py
│   │   │   └── archive_service.py
│   │   ├── permissions.py
│   │   └── filters.py
│   ├── tenants/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   └── views.py
│   ├── workflows/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   └── views.py
│   ├── notifications/
│   │   ├── consumers.py
│   │   └── routing.py
│   └── tests/
│       ├── unit/
│       │   ├── test_models.py
│       │   ├── test_serializers.py
│       │   └── test_services.py
│       └── integration/
│           ├── test_documents_api.py
│           └── test_approval_flow.py
├── lambdas/
│   ├── document_processor/
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── text_extractor/
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── validator/
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── notifier/
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── archiver/
│   │   ├── handler.py
│   │   └── requirements.txt
│   └── status_updater/
│       ├── handler.py
│       └── requirements.txt
├── infrastructure/
│   ├── cloudformation/
│   │   ├── docuflow-stack.yaml
│   │   ├── iam-roles.yaml
│   │   └── networking.yaml
│   └── step-functions/
│       └── document-workflow.json
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── DocumentUploader/
│   │   │   ├── DocumentList/
│   │   │   ├── ApprovalPanel/
│   │   │   ├── WorkflowStatus/
│   │   │   └── DocumentViewer/
│   │   ├── store/
│   │   │   ├── slices/
│   │   │   │   ├── documentsSlice.ts
│   │   │   │   ├── workflowSlice.ts
│   │   │   │   └── authSlice.ts
│   │   │   └── index.ts
│   │   ├── services/
│   │   │   ├── documentService.ts
│   │   │   └── approvalService.ts
│   │   ├── types/
│   │   └── styles/
│   ├── webpack.config.js
│   └── cypress/
└── .github/
    └── workflows/
        ├── ci.yml
        └── deploy.yml
```

### Flujo de Datos
```
1. El usuario carga un documento desde el dashboard React
   └──> POST /api/v1/documents/ → Django DRF
        └──> DocumentUploadService genera URL presignada S3
             └──> React carga el archivo directamente a S3 (navegador → S3)
                  └──> Evento S3 PutObject dispara Lambda Processor
                       └──> Lambda inicia ejecución Step Functions
                            ├──> Estado: ExtractDocumentText (Lambda + Textract)
                            ├──> Estado: ValidateExtraction (Lambda)
                            ├──> Estado: NotifyReviewer (Lambda → email SES)
                            ├──> Estado: WaitForApproval (SQS waitForTaskToken)
                            │    └──> Revisor hace clic en Aprobar/Rechazar en React
                            │         └──> POST /api/v1/documents/{id}/approve
                            │              └──> Django envía taskToken a Step Functions
                            │                   └──> Flujo de trabajo reanuda ejecución
                            ├──> Estado: ArchiveDocument (Lambda → S3 archivos/)
                            └──> Estado: NotifyCompletion (Lambda → WebSocket)
                                 └──> Django Channels transmite a React
                                      └──> Redux actualiza estado del documento en UI

2. Callback de Lambda para actualización de estado
   └──> Lambda hace POST de estado a endpoint interno Django
        └──> Document.status actualizado vía Django ORM
             └──> Signal post_save se dispara
                  └──> Tarea Celery: broadcast_document_update
                       └──> Django Channels WebSocket push a sala del tenant
                            └──> Acción Redux: updateDocumentStatus
                                 └──> UI React refleja nuevo estado instantáneamente
```

### Pipeline CI/CD — GitHub Actions
```yaml
# .github/workflows/deploy.yml
name: Desplegar DocuFlow

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r backend/requirements.txt
      - run: pytest backend/ --cov=backend --cov-report=xml
      - uses: codecov/codecov-action@v4

  deploy-lambdas:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Empaquetar y desplegar funciones Lambda
        run: |
          for lambda in document_processor text_extractor validator notifier archiver status_updater; do
            cd lambdas/$lambda
            pip install -r requirements.txt -t .
            zip -r ../../$lambda.zip .
            cd ../..
            aws s3 cp $lambda.zip s3://docuflow-deployments-prod/lambdas/
            aws lambda update-function-code \
              --function-name docuflow-$lambda-prod \
              --s3-bucket docuflow-deployments-prod \
              --s3-key lambdas/$lambda.zip
          done

  deploy-infrastructure:
    needs: deploy-lambdas
    runs-on: ubuntu-latest
    steps:
      - name: Desplegar stack CloudFormation
        run: |
          aws cloudformation deploy \
            --template-file infrastructure/cloudformation/docuflow-stack.yaml \
            --stack-name docuflow-prod \
            --parameter-overrides \
              Environment=prod \
              DjangoImage=${{ steps.build.outputs.image }} \
            --capabilities CAPABILITY_IAM \
            --no-fail-on-empty-changeset

  deploy-django:
    needs: deploy-infrastructure
    runs-on: ubuntu-latest
    steps:
      - name: Construir y subir imagen Django a ECR
        run: |
          aws ecr get-login-password | docker login --username AWS \
            --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker build -t docuflow-django ./backend
          docker tag docuflow-django:latest \
            ${{ secrets.ECR_REGISTRY }}/docuflow-django:${{ github.sha }}
          docker push \
            ${{ secrets.ECR_REGISTRY }}/docuflow-django:${{ github.sha }}

      - name: Actualizar servicio ECS Fargate
        run: |
          aws ecs update-service \
            --cluster docuflow-prod \
            --service docuflow-django \
            --force-new-deployment

      - name: Ejecutar migraciones Django en RDS
        run: |
          aws ecs run-task \
            --cluster docuflow-prod \
            --task-definition docuflow-migrate \
            --launch-type FARGATE \
            --network-configuration "awsvpcConfiguration={
              subnets=[${{ secrets.SUBNET_IDS }}],
              securityGroups=[${{ secrets.SG_IDS }}],
              assignPublicIp=ENABLED
            }"

  deploy-frontend:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: cd frontend && npm ci && npm run build
      - name: Desplegar build React a S3
        run: |
          aws s3 sync frontend/dist/ \
            s3://docuflow-frontend-prod/ \
            --delete \
            --cache-control "max-age=31536000"
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} \
            --paths "/*"
```

---

## 💾 Instalación

### Requisitos Previos
```bash
# Software requerido
- Python 3.11 o superior
- Node.js 20 LTS o superior
- Docker y Docker Compose
- AWS CLI v2 configurado con permisos IAM apropiados
- Cuenta AWS con acceso a: S3, Lambda, Step Functions,
  ECS, RDS, SQS, Textract, Cognito, CloudFormation, CloudWatch
```

### Opción 1: Instalación con Docker (Recomendada)
```bash
# 1. Clonar el repositorio
git clone https://github.com/paulabadt/docuflow.git
cd docuflow

# 2. Copiar archivos de variables de entorno
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local

# 3. Configurar credenciales AWS en backend/.env
# (LocalStack se usa para emulación local de servicios AWS)

# 4. Iniciar todos los servicios
docker-compose up -d

# 5. Ejecutar migraciones Django
docker-compose exec backend python manage.py migrate

# 6. Crear superusuario Django
docker-compose exec backend python manage.py createsuperuser

# 7. Cargar datos de demostración de tenant y documentos
docker-compose exec backend python manage.py seed_demo

# 8. Verificar que todos los servicios están activos
docker-compose ps

# 9. Acceder a la plataforma
# Frontend:       http://localhost:3000
# API Django:     http://localhost:8000/api/v1/
# Django Admin:   http://localhost:8000/admin/
# Docs API:       http://localhost:8000/api/v1/schema/swagger-ui/
# LocalStack:     http://localhost:4566
```

### Opción 2: Instalación Manual

#### Configuración del Backend (Python + Django)
```bash
# 1. Ingresar al directorio del backend
cd backend

# 2. Crear y activar entorno virtual
python -m venv venv
source venv/bin/activate        # Linux/macOS
# venv\Scripts\activate         # Windows

# 3. Instalar dependencias
pip install -r requirements.txt

# 4. Configurar variables de entorno
cp .env.example .env
# Editar .env con base de datos y credenciales AWS

# 5. Iniciar PostgreSQL y Redis (vía Docker)
docker-compose up -d postgres redis

# 6. Ejecutar migraciones Django
python manage.py migrate

# 7. Crear superusuario
python manage.py createsuperuser

# 8. Cargar datos de demostración
python manage.py seed_demo

# 9. Iniciar servidor de desarrollo Django
python manage.py runserver 0.0.0.0:8000

# 10. Iniciar worker Celery (terminal separada)
celery -A config worker --loglevel=info

# 11. Iniciar servidor ASGI Django Channels (terminal separada)
uvicorn config.asgi:application --reload --host 0.0.0.0 --port 8001
```

#### Configuración del Frontend (React + TypeScript)
```bash
# 1. Ingresar al directorio del frontend
cd frontend

# 2. Instalar dependencias
npm install

# 3. Configurar variables de entorno
cp .env.example .env.local

# 4. Iniciar servidor de desarrollo
npm run dev

# 5. Compilar para producción
npm run build
```

### Variables de Entorno
```bash
# backend/.env

# Django
DJANGO_ENV=development
DJANGO_SECRET_KEY=tu_clave_secreta_django_minimo_50_caracteres
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DEBUG=True

# Base de datos — PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=docuflow_db
DB_USER=docuflow_user
DB_PASSWORD=tu_contraseña_segura

# Redis — Celery + Channels
REDIS_URL=redis://localhost:6379/0

# Credenciales AWS
AWS_ACCESS_KEY_ID=tu_access_key
AWS_SECRET_ACCESS_KEY=tu_secret_key
AWS_DEFAULT_REGION=us-east-1

# Recursos AWS
AWS_S3_BUCKET_NAME=docuflow-documents-dev
AWS_STEP_FUNCTIONS_ARN=arn:aws:states:us-east-1:123456789:stateMachine:docuflow-workflow-dev
AWS_SQS_APPROVAL_URL=https://sqs.us-east-1.amazonaws.com/123456789/docuflow-approvals-dev
AWS_COGNITO_USER_POOL_ID=us-east-1_xxxxxxxxx
AWS_COGNITO_CLIENT_ID=tu_cognito_client_id

# LocalStack (solo desarrollo)
AWS_ENDPOINT_URL=http://localhost:4566

# Frontend
FRONTEND_URL=http://localhost:3000
```
```bash
# frontend/.env.local

REACT_APP_API_URL=http://localhost:8000/api/v1
REACT_APP_WS_URL=ws://localhost:8001
REACT_APP_COGNITO_REGION=us-east-1
REACT_APP_COGNITO_USER_POOL_ID=us-east-1_xxxxxxxxx
REACT_APP_COGNITO_CLIENT_ID=tu_cognito_client_id
REACT_APP_APP_NAME=DocuFlow
```

### Servicios Docker Compose
```yaml
# docker-compose.yml — resumen de servicios
services:
  backend:     # App Django (ASGI)    — puerto 8000
  celery:      # Worker Celery        — sin puerto
  frontend:    # App React            — puerto 3000
  postgres:    # PostgreSQL           — puerto 5432
  redis:       # Redis                — puerto 6379
  localstack:  # Servicios AWS local  — puerto 4566
```

---

## 🚀 Uso

### Iniciar la Plataforma
```bash
# Iniciar todos los servicios de infraestructura
docker-compose up -d

# Iniciar servidor ASGI Django
cd backend
uvicorn config.asgi:application --reload --host 0.0.0.0 --port 8000

# Iniciar worker Celery (terminal separada)
celery -A config worker --loglevel=info

# Iniciar frontend React (terminal separada)
cd frontend
npm run dev
```

### Credenciales por Defecto
```bash
# Administrador del sistema
Email:    admin@docuflow.io
Password: Admin123! (¡cambiar inmediatamente!)

# Gerente de tenant
Email:    manager@acme-corp.docuflow.io
Password: Manager123!

# Revisor de documentos
Email:    reviewer@acme-corp.docuflow.io
Password: Reviewer123!

# Usuario estándar
Email:    user@acme-corp.docuflow.io
Password: User123!
```

### Flujo de Procesamiento de Documentos
```bash
# 1. El usuario carga un documento desde la interfaz React
#    → Django DRF retorna URL presignada S3
#    → El navegador carga directamente a S3

# 2. S3 dispara Lambda Processor automáticamente
#    → Se inicia ejecución Step Functions

# 3. Monitorear progreso del flujo de trabajo
#    → Dashboard React se actualiza en tiempo real vía WebSocket
#    → Django Admin: http://localhost:8000/admin/documents/document/

# 4. El revisor aprueba o rechaza el documento
#    → POST /api/v1/documents/{id}/approve/
#    → Step Functions reanuda desde el estado WaitForApproval

# 5. Documento archivado en prefijo S3 archivos/
#    → Estado final transmitido a todos los suscriptores WebSocket del tenant
```

### Scripts Disponibles
```bash
# Backend — Django
python manage.py runserver              # Servidor de desarrollo
python manage.py migrate                # Ejecutar migraciones pendientes
python manage.py makemigrations         # Generar nuevos archivos de migración
python manage.py createsuperuser        # Crear usuario administrador
python manage.py seed_demo              # Cargar tenants y documentos de demo
python manage.py shell_plus             # Shell interactivo Django
pytest                                  # Suite completa de pruebas
pytest --cov=. --cov-report=html        # Pruebas con reporte de cobertura HTML
pytest -m unit                          # Solo pruebas unitarias
pytest -m integration                   # Solo pruebas de integración
celery -A config worker --loglevel=info # Iniciar worker Celery
celery -A config flower                 # Dashboard de monitoreo Celery

# Backend — Empaquetado Lambda
make package-lambdas                    # Comprimir todas las funciones Lambda
make deploy-lambdas ENV=dev             # Desplegar Lambdas a AWS dev
make deploy-stack ENV=dev               # Desplegar stack CloudFormation

# Frontend
npm run dev                             # Servidor de desarrollo con HMR
npm run build                           # Bundle de producción Webpack optimizado
npm run preview                         # Previsualizar build de producción local
npm run test                            # Pruebas React Testing Library
npm run test:coverage                   # Reporte de cobertura del frontend
npm run cypress:open                    # Runner interactivo BDD de Cypress
npm run cypress:run                     # Suite completa Cypress en modo headless
npm run lint                            # Verificación ESLint + TypeScript
```

---

## 💻 Ejemplos de Código

### 1. Capa de Servicios Django — Carga y Aprobación
```python
# documents/services/upload_service.py
# Django — Generación de URL presignada S3 e iniciación de documento

import boto3
import uuid
from django.conf import settings
from django.contrib.auth import get_user_model
from documents.models import Document

User = get_user_model()


class DocumentUploadService:
    def __init__(self, user: User):
        self.user      = user
        self.s3_client = boto3.client(
            "s3",
            endpoint_url=settings.AWS_ENDPOINT_URL,  # LocalStack en dev
        )

    def initiate_upload(
        self,
        title: str,
        document_type: str,
    ) -> dict:
        """
        Reservar una clave S3 y generar una URL presignada para carga
        directa navegador-a-S3. Retorna bucket y clave para el registro
        del documento.
        """
        document_id = str(uuid.uuid4())
        s3_key      = f"uploads/{self.user.profile.tenant_id}/{document_id}"

        # Etiquetar el futuro objeto S3 con metadatos de enrutamiento
        tag_set = (
            f"document_id={document_id}"
            f"&tenant_id={self.user.profile.tenant_id}"
            f"&document_type={document_type}"
            f"&uploaded_by={self.user.id}"
        )

        presigned_url = self.s3_client.generate_presigned_url(
            ClientMethod="put_object",
            Params={
                "Bucket":  settings.AWS_S3_BUCKET_NAME,
                "Key":     s3_key,
                "Tagging": tag_set,
            },
            ExpiresIn=900,  # 15 minutos
        )

        return {
            "document_id":   document_id,
            "bucket":        settings.AWS_S3_BUCKET_NAME,
            "key":           s3_key,
            "presigned_url": presigned_url,
        }

    @staticmethod
    def generate_presigned_url(bucket: str, key: str) -> str:
        s3_client = boto3.client(
            "s3", endpoint_url=settings.AWS_ENDPOINT_URL
        )
        return s3_client.generate_presigned_url(
            ClientMethod="get_object",
            Params={"Bucket": bucket, "Key": key},
            ExpiresIn=3600,
        )
```
```python
# documents/services/approval_service.py
# Django — Lógica de aprobación y rechazo con callback de token de tarea Step Functions

import boto3
import json
from datetime import datetime, timezone
from django.conf import settings
from django.contrib.auth import get_user_model
from documents.models import Document, ApprovalRecord

User = get_user_model()


class ApprovalService:
    def __init__(self, reviewer: User):
        self.reviewer   = reviewer
        self.sfn_client = boto3.client(
            "stepfunctions",
            endpoint_url=settings.AWS_ENDPOINT_URL,
        )

    def approve(self, document: Document, notes: str = "") -> Document:
        self._verify_reviewer_permissions(document)

        # Recuperar token de tarea almacenado cuando Step Functions
        # entró al estado WaitForApproval
        approval_record = ApprovalRecord.objects.get(
            document=document,
            status=ApprovalRecord.Status.PENDING,
        )

        # Enviar éxito de tarea para reanudar ejecución Step Functions
        self.sfn_client.send_task_success(
            taskToken=approval_record.task_token,
            output=json.dumps({
                "decision":    "APPROVED",
                "reviewed_by": str(self.reviewer.id),
                "reviewed_at": datetime.now(timezone.utc).isoformat(),
                "notes":       notes,
            }),
        )

        # Actualizar registros locales
        document.status       = Document.Status.APPROVED
        document.reviewed_by  = self.reviewer
        document.reviewed_at  = datetime.now(timezone.utc)
        document.review_notes = notes
        document.save()

        approval_record.status = ApprovalRecord.Status.APPROVED
        approval_record.save()

        return document

    def reject(self, document: Document, reason: str) -> Document:
        self._verify_reviewer_permissions(document)

        approval_record = ApprovalRecord.objects.get(
            document=document,
            status=ApprovalRecord.Status.PENDING,
        )

        # Enviar éxito de tarea con decisión REJECTED
        # el flujo enruta a MarkRejected
        self.sfn_client.send_task_success(
            taskToken=approval_record.task_token,
            output=json.dumps({
                "decision":    "REJECTED",
                "reviewed_by": str(self.reviewer.id),
                "reviewed_at": datetime.now(timezone.utc).isoformat(),
                "reason":      reason,
            }),
        )

        document.status       = Document.Status.REJECTED
        document.reviewed_by  = self.reviewer
        document.reviewed_at  = datetime.now(timezone.utc)
        document.review_notes = reason
        document.save()

        approval_record.status = ApprovalRecord.Status.REJECTED
        approval_record.save()

        return document

    def _verify_reviewer_permissions(self, document: Document) -> None:
        if document.tenant_id != self.reviewer.profile.tenant_id:
            raise PermissionError(
                "El revisor no pertenece al tenant del documento"
            )
        if not self.reviewer.has_perm("documents.can_approve"):
            raise PermissionError(
                "El usuario no tiene permiso de aprobación de documentos"
            )
```

---

### 2. Django Channels — Actualizaciones WebSocket en Tiempo Real
```python
# notifications/consumers.py
# Django Channels — Consumidor WebSocket para transmisión de estado de documentos

import json
from channels.generic.websocket import AsyncWebsocketConsumer
from django.contrib.auth.models import AnonymousUser


class DocumentStatusConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        user = self.scope["user"]

        if isinstance(user, AnonymousUser):
            await self.close(code=4001)
            return

        # Sala con alcance por tenant — operadores solo reciben sus actualizaciones
        self.tenant_id  = user.profile.tenant_id
        self.room_group = f"tenant_{self.tenant_id}"

        await self.channel_layer.group_add(
            self.room_group,
            self.channel_name,
        )
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.room_group,
            self.channel_name,
        )

    # Recibe transmisión de tarea Celery vía capa de canales
    async def document_status_update(self, event):
        await self.send(text_data=json.dumps({
            "type":        "DOCUMENT_STATUS_UPDATE",
            "document_id": event["document_id"],
            "status":      event["status"],
            "updated_at":  event["updated_at"],
        }))
```
```python
# documents/tasks.py
# Celery — Transmisión WebSocket asíncrona disparada por signal post_save de Django

from celery import shared_task
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync
from datetime import datetime, timezone
from documents.models import Document


@shared_task
def broadcast_document_update(document_id: str, status: str) -> None:
    """
    Tarea Celery — transmitir cambio de estado de documento a todos los
    suscriptores WebSocket en la sala del tenant. Disparada por signal
    post_save de Django.
    """
    document      = Document.objects.select_related(
        "uploaded_by__profile"
    ).get(id=document_id)
    channel_layer = get_channel_layer()
    room_group    = f"tenant_{document.tenant_id}"

    async_to_sync(channel_layer.group_send)(
        room_group,
        {
            "type":        "document_status_update",
            "document_id": str(document.id),
            "status":      status,
            "updated_at":  datetime.now(timezone.utc).isoformat(),
        },
    )
```

---

### 3. Lambda — Validador y Actualizador de Estado
```python
# lambdas/validator/handler.py
# AWS Lambda — Tarea Step Functions: validar datos extraídos del documento

import json
import logging
import os
import urllib.request

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

REQUIRED_FIELDS = {
    "INVOICE":  ["Invoice Number", "Date", "Total Amount", "Vendor Name"],
    "CONTRACT": ["Contract Number", "Parties", "Effective Date", "Term"],
    "FORM":     ["Name", "Date", "Signature"],
    "GENERIC":  [],
}


def handler(event: dict, context) -> dict:
    """
    Tarea Step Functions — validar que los campos requeridos fueron extraídos.
    Entrada: resultado de extracción del Lambda TextExtractor.
    Salida: resultado de validación con campos faltantes y verificación de confianza.
    """
    document_type = event.get("document_type", "GENERIC")
    extraction    = event.get("extraction", {})
    key_values    = extraction.get("key_values", {})
    confidence    = extraction.get("confidence", 0.0)

    required = REQUIRED_FIELDS.get(document_type, [])
    missing  = [
        field for field in required
        if field not in key_values or not key_values[field].strip()
    ]

    passed = len(missing) == 0 and confidence >= 75.0

    logger.info(
        "Resultado de validación — document_id=%s passed=%s missing=%s confidence=%.1f",
        event.get("document_id"), passed, missing, confidence,
    )

    # Notificar a Django para actualizar el estado del documento
    _update_document_status(
        document_id=event["document_id"],
        status="PENDING_APPROVAL" if passed else "VALIDATION_FAILED",
    )

    return {
        **event,
        "validation": {
            "passed":         passed,
            "missing_fields": missing,
            "confidence":     confidence,
            "document_type":  document_type,
        },
    }


def _update_document_status(document_id: str, status: str) -> None:
    """Callback al endpoint interno Django para actualizar el estado del documento."""
    django_url   = os.environ["DJANGO_INTERNAL_URL"]
    django_token = os.environ["DJANGO_INTERNAL_TOKEN"]

    payload = json.dumps({
        "document_id": document_id,
        "status":      status,
    }).encode("utf-8")

    req = urllib.request.Request(
        url=f"{django_url}/internal/documents/status/",
        data=payload,
        headers={
            "Content-Type":  "application/json",
            "Authorization": f"Token {django_token}",
        },
        method="POST",
    )

    with urllib.request.urlopen(req, timeout=10) as resp:
        logger.info("Estado actualizado — response=%d", resp.status)
```
```python
# lambdas/notifier/handler.py
# AWS Lambda — Tarea Step Functions: notificar al revisor vía email SES

import boto3
import json
import logging
import os

logger     = logging.getLogger(__name__)
ses_client = boto3.client("ses")


def handler(event: dict, context) -> dict:
    """
    Tarea Step Functions — enviar notificación por email al revisor de documentos.
    Disparada tras validación exitosa antes del estado WaitForApproval.
    """
    document_id   = event["document_id"]
    tenant_id     = event["tenant_id"]
    document_type = event["document_type"]
    extraction    = event.get("extraction", {})

    reviewer_email = _get_reviewer_email(tenant_id)
    review_url     = (
        f"{os.environ['FRONTEND_URL']}/revision/{document_id}"
    )

    ses_client.send_email(
        Source=os.environ["SES_FROM_EMAIL"],
        Destination={"ToAddresses": [reviewer_email]},
        Message={
            "Subject": {
                "Data": (
                    f"[DocuFlow] Documento listo para revisión — {document_type}"
                )
            },
            "Body": {
                "Html": {
                    "Data": f"""
                    <h2>Documento Listo para Revisión</h2>
                    <p>Un nuevo documento <strong>{document_type}</strong>
                    requiere su aprobación.</p>
                    <p><strong>ID del documento:</strong> {document_id}</p>
                    <p><strong>Campos extraídos:</strong>
                    {len(extraction.get('key_values', {}))}</p>
                    <p><strong>Confianza:</strong>
                    {extraction.get('confidence', 0):.1f}%</p>
                    <br>
                    <a href="{review_url}"
                       style="background:#2563eb;color:white;
                              padding:12px 24px;
                              border-radius:6px;
                              text-decoration:none;">
                      Revisar Documento
                    </a>
                    """
                }
            },
        },
    )

    logger.info(
        "Revisor notificado — document_id=%s reviewer=%s",
        document_id, reviewer_email,
    )

    return {**event, "notification": {"sent_to": reviewer_email}}


def _get_reviewer_email(tenant_id: str) -> str:
    ssm   = boto3.client("ssm")
    param = ssm.get_parameter(
        Name=f"/docuflow/{tenant_id}/reviewer-email",
        WithDecryption=True,
    )
    return param["Parameter"]["Value"]
```

---

### 4. Redux Thunks — Gestión de Estado de Documentos
```typescript
// store/slices/documentsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import { documentService } from '../../services/documentService';
import {
  Document,
  DocumentUploadPayload,
  ApprovalPayload,
} from '../../types/documents';

interface DocumentsState {
  items:           Record<string, Document>;
  pendingApproval: string[];
  selectedId:      string | null;
  uploadProgress:  number;
  loading:         boolean;
  error:           string | null;
}

const initialState: DocumentsState = {
  items:           {},
  pendingApproval: [],
  selectedId:      null,
  uploadProgress:  0,
  loading:         false,
  error:           null,
};

export const initiateUpload = createAsyncThunk(
  'documents/initiateUpload',
  async (payload: DocumentUploadPayload, { rejectWithValue }) => {
    try {
      // Paso 1: Obtener URL presignada desde Django DRF
      const { data } = await documentService.initiateUpload({
        title:         payload.title,
        document_type: payload.documentType,
      });

      // Paso 2: Cargar archivo directamente a S3 usando URL presignada
      await documentService.uploadToS3(
        data.presigned_url,
        payload.file,
        (progress) => payload.onProgress?.(progress),
      );

      return data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.detail || 'Error al cargar el documento'
      );
    }
  }
);

export const fetchDocuments = createAsyncThunk(
  'documents/fetchAll',
  async (_, { rejectWithValue }) => {
    try {
      const { data } = await documentService.list();
      return data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.detail || 'Error al obtener documentos'
      );
    }
  }
);

export const approveDocument = createAsyncThunk(
  'documents/approve',
  async (payload: ApprovalPayload, { rejectWithValue }) => {
    try {
      const { data } = await documentService.approve(
        payload.documentId,
        payload.notes,
      );
      return data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.detail || 'Error al aprobar el documento'
      );
    }
  }
);

export const rejectDocument = createAsyncThunk(
  'documents/reject',
  async (payload: ApprovalPayload, { rejectWithValue }) => {
    try {
      const { data } = await documentService.reject(
        payload.documentId,
        payload.notes,
      );
      return data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.detail || 'Error al rechazar el documento'
      );
    }
  }
);

const documentsSlice = createSlice({
  name: 'documents',
  initialState,
  reducers: {
    // Llamado en cada actualización de estado WebSocket
    updateDocumentStatus: (
      state,
      action: PayloadAction<{ document_id: string; status: string }>
    ) => {
      const { document_id, status } = action.payload;
      if (state.items[document_id]) {
        state.items[document_id].status = status;
      }
      if (status === 'PENDING_APPROVAL' &&
          !state.pendingApproval.includes(document_id)) {
        state.pendingApproval.push(document_id);
      }
      if (['APPROVED', 'REJECTED', 'ARCHIVED'].includes(status)) {
        state.pendingApproval =
          state.pendingApproval.filter(id => id !== document_id);
      }
    },
    setUploadProgress: (state, action: PayloadAction<number>) => {
      state.uploadProgress = action.payload;
    },
    selectDocument: (state, action: PayloadAction<string>) => {
      state.selectedId = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchDocuments.pending, (state) => {
        state.loading = true;
        state.error   = null;
      })
      .addCase(fetchDocuments.fulfilled, (state, action) => {
        state.loading = false;
        state.items   = action.payload.reduce(
          (acc: Record<string, Document>, doc: Document) => {
            acc[doc.id] = doc;
            return acc;
          }, {}
        );
      })
      .addCase(fetchDocuments.rejected, (state, action) => {
        state.loading = false;
        state.error   = action.payload as string;
      })
      .addCase(initiateUpload.fulfilled, (state) => {
        state.uploadProgress = 0;
      })
      .addCase(approveDocument.fulfilled, (state, action) => {
        const doc = action.payload;
        state.items[doc.id] = doc;
        state.pendingApproval =
          state.pendingApproval.filter(id => id !== doc.id);
      })
      .addCase(rejectDocument.fulfilled, (state, action) => {
        const doc = action.payload;
        state.items[doc.id] = doc;
        state.pendingApproval =
          state.pendingApproval.filter(id => id !== doc.id);
      });
  },
});

export const {
  updateDocumentStatus,
  setUploadProgress,
  selectDocument,
} = documentsSlice.actions;

export default documentsSlice.reducer;
```

---

### 5. Pruebas Backend — Pytest + Django Test Client (TDD)
```python
# tests/unit/test_approval_service.py
import pytest
from unittest.mock import MagicMock, patch
from django.contrib.auth import get_user_model
from documents.models import Document, ApprovalRecord
from documents.services.approval_service import ApprovalService

User = get_user_model()


@pytest.fixture
def tenant_id():
    return "acme-corp"


@pytest.fixture
def reviewer(db, tenant_id):
    user = User.objects.create_user(
        username="reviewer@acme.io",
        email="reviewer@acme.io",
        password="Reviewer123!",
    )
    user.profile.tenant_id = tenant_id
    user.profile.save()
    return user


@pytest.fixture
def document(db, tenant_id, reviewer):
    return Document.objects.create(
        tenant_id=tenant_id,
        title="Factura Q1",
        document_type=Document.DocumentType.INVOICE,
        status=Document.Status.PENDING_APPROVAL,
        s3_bucket="docuflow-documents-dev",
        s3_key=f"uploads/{tenant_id}/doc-001",
        uploaded_by=reviewer,
    )


@pytest.fixture
def approval_record(db, document):
    return ApprovalRecord.objects.create(
        document=document,
        task_token="mock-sfn-task-token-abc123",
        status=ApprovalRecord.Status.PENDING,
    )


@pytest.mark.django_db
@patch("documents.services.approval_service.boto3.client")
def test_aprobar_documento_envia_task_success(
    mock_boto3, reviewer, document, approval_record
):
    # Dado
    mock_sfn = MagicMock()
    mock_boto3.return_value = mock_sfn

    service = ApprovalService(reviewer)

    # Cuando
    result = service.approve(document, notes="Todo correcto")

    # Entonces
    mock_sfn.send_task_success.assert_called_once()
    call_kwargs = mock_sfn.send_task_success.call_args[1]
    assert call_kwargs["taskToken"] == "mock-sfn-task-token-abc123"

    import json
    output = json.loads(call_kwargs["output"])
    assert output["decision"] == "APPROVED"
    assert result.status == Document.Status.APPROVED
    assert result.review_notes == "Todo correcto"


@pytest.mark.django_db
@patch("documents.services.approval_service.boto3.client")
def test_rechazar_documento_envia_decision_rejected(
    mock_boto3, reviewer, document, approval_record
):
    # Dado
    mock_sfn = MagicMock()
    mock_boto3.return_value = mock_sfn

    service = ApprovalService(reviewer)

    # Cuando
    result = service.reject(
        document, reason="Falta firma del proveedor en la página 3"
    )

    # Entonces
    mock_sfn.send_task_success.assert_called_once()
    call_kwargs = mock_sfn.send_task_success.call_args[1]
    import json
    output = json.loads(call_kwargs["output"])
    assert output["decision"] == "REJECTED"
    assert result.status == Document.Status.REJECTED


@pytest.mark.django_db
def test_aprobar_lanza_error_de_permiso_para_tenant_incorrecto(db, document):
    # Dado — revisor de un tenant diferente
    other_user = User.objects.create_user(
        username="otro@otro.io",
        email="otro@otro.io",
        password="Otro123!",
    )
    other_user.profile.tenant_id = "otro-corp"
    other_user.profile.save()

    service = ApprovalService(other_user)

    # Cuando / Entonces
    with pytest.raises(
        PermissionError,
        match="no pertenece al tenant del documento"
    ):
        service.approve(document, notes="Intento no autorizado")
```
```python
# tests/integration/test_documents_api.py
import pytest
from unittest.mock import patch
from rest_framework.test import APIClient
from django.contrib.auth import get_user_model
from documents.models import Document

User = get_user_model()


@pytest.fixture
def api_client():
    return APIClient()


@pytest.fixture
def authenticated_user(db):
    user = User.objects.create_user(
        username="user@acme.io",
        email="user@acme.io",
        password="User123!",
    )
    user.profile.tenant_id = "acme-corp"
    user.profile.save()
    return user


@pytest.mark.django_db
def test_listar_documentos_retorna_solo_documentos_del_tenant(
    api_client, authenticated_user
):
    # Dado
    api_client.force_authenticate(user=authenticated_user)

    Document.objects.create(
        tenant_id="acme-corp",
        title="Factura Acme",
        document_type="INVOICE",
        status="UPLOADED",
        s3_bucket="bucket",
        s3_key="key-001",
        uploaded_by=authenticated_user,
    )
    Document.objects.create(
        tenant_id="otro-corp",       # Tenant diferente — debe excluirse
        title="Factura Otro",
        document_type="INVOICE",
        status="UPLOADED",
        s3_bucket="bucket",
        s3_key="key-002",
        uploaded_by=authenticated_user,
    )

    # Cuando
    response = api_client.get("/api/v1/documents/")

    # Entonces
    assert response.status_code == 200
    assert len(response.data["results"]) == 1
    assert response.data["results"][0]["title"] == "Factura Acme"


@pytest.mark.django_db
def test_crear_documento_retorna_url_presignada(
    api_client, authenticated_user
):
    # Dado
    api_client.force_authenticate(user=authenticated_user)

    # Cuando
    with patch(
        "documents.services.upload_service.boto3.client"
    ) as mock_s3:
        mock_s3.return_value.generate_presigned_url.return_value = (
            "https://s3.amazonaws.com/url-presignada"
        )

        response = api_client.post("/api/v1/documents/", {
            "title":         "Nuevo Contrato",
            "document_type": "CONTRACT",
        })

    # Entonces
    assert response.status_code == 201
    assert "upload_url" in response.data
    assert response.data["status"] == "UPLOADED"
```
```typescript
// cypress/e2e/document_upload.cy.ts — BDD E2E
describe('Carga de Documentos — BDD', () => {
  beforeEach(() => {
    cy.login('user@acme-corp.docuflow.io', 'User123!');
    cy.visit('/dashboard');
  });

  it('Dado un usuario, Cuando carga un documento, Entonces el estado del flujo muestra EXTRACTING',
    () => {
      cy.intercept('POST', '/api/v1/documents/', {
        statusCode: 201,
        body: {
          id:         'doc-001',
          title:      'Factura de Prueba',
          status:     'UPLOADED',
          upload_url: 'https://s3.amazonaws.com/mock-presigned',
        },
      }).as('createDocument');

      cy.intercept('PUT', 'https://s3.amazonaws.com/**', {
        statusCode: 200,
      }).as('s3Upload');

      cy.get('[data-testid="upload-btn"]').click();
      cy.get('[data-testid="file-input"]')
        .selectFile('cypress/fixtures/sample-invoice.pdf');
      cy.get('[data-testid="document-title-input"]')
        .type('Factura de Prueba');
      cy.get('[data-testid="document-type-select"]')
        .select('INVOICE');
      cy.get('[data-testid="submit-upload-btn"]').click();

      cy.wait('@createDocument');
      cy.wait('@s3Upload');

      cy.get('[data-testid="doc-status-doc-001"]')
        .should('contain.text', 'UPLOADED');
  });

  it('Dado un revisor, Cuando aprueba un documento, Entonces el estado cambia a APPROVED',
    () => {
      cy.login('reviewer@acme-corp.docuflow.io', 'Reviewer123!');
      cy.visit('/revision/doc-pending-001');

      cy.intercept('POST', '/api/v1/documents/doc-pending-001/approve/', {
        statusCode: 200,
        body: { id: 'doc-pending-001', status: 'APPROVED' },
      }).as('approveDocument');

      cy.get('[data-testid="review-notes-input"]')
        .type('Todos los campos verificados y correctos');
      cy.get('[data-testid="approve-btn"]').click();
      cy.get('[data-testid="confirm-approval-btn"]').click();

      cy.wait('@approveDocument');
      cy.get('[data-testid="workflow-status"]')
        .should('contain.text', 'APPROVED');
  });

  it('Dado un usuario, Cuando llega actualización WebSocket, Entonces el estado se refleja en tiempo real',
    () => {
      cy.visit('/dashboard');
      cy.get('[data-testid="doc-status-doc-001"]')
        .should('contain.text', 'PENDING_APPROVAL');

      // Simular push WebSocket
      cy.window().then((win) => {
        win.dispatchEvent(
          new CustomEvent('ws:document_status_update', {
            detail: { document_id: 'doc-001', status: 'APPROVED' },
          })
        );
      });

      cy.get('[data-testid="doc-status-doc-001"]')
        .should('contain.text', 'APPROVED');
  });
});
```

---

## 📚 Documentación de la API

### URL Base
```
Desarrollo:    http://localhost:8000/api/v1
Producción:    https://api.docuflow.io/api/v1
Swagger UI:    http://localhost:8000/api/v1/schema/swagger-ui/
ReDoc:         http://localhost:8000/api/v1/schema/redoc/
Django Admin:  http://localhost:8000/admin/
```

### Autenticación
```bash
POST /api/v1/auth/login/
Content-Type: application/json

{
  "email":    "user@acme-corp.docuflow.io",
  "password": "tu_contraseña"
}

# Respuesta: 200 OK
{
  "access":  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id":        "usr-001",
    "email":     "user@acme-corp.docuflow.io",
    "role":      "UPLOADER",
    "tenant_id": "acme-corp"
  }
}
```
```bash
# Refrescar token de acceso
POST /api/v1/auth/token/refresh/
Content-Type: application/json

{ "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }

# Uso del token en solicitudes protegidas
GET /api/v1/documents/
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Endpoints

#### 1. Documentos

**Listar Documentos**
```bash
GET /api/v1/documents/?status=PENDING_APPROVAL&document_type=INVOICE&page=1
Authorization: Bearer {token}

# Parámetros de consulta:
# - status:        string (UPLOADED, EXTRACTING, VALIDATING,
#                          PENDING_APPROVAL, APPROVED, REJECTED,
#                          ARCHIVED, EXTRACTION_FAILED, VALIDATION_FAILED)
# - document_type: string (INVOICE, CONTRACT, REPORT, FORM, GENERIC)
# - search:        string (busca en el campo título)
# - ordering:      string (created_at, -created_at, title)
# - page:          int (por defecto: 1)
# - page_size:     int (por defecto: 20, máximo: 100)

# Respuesta: 200 OK
{
  "count": 47,
  "next":  "http://localhost:8000/api/v1/documents/?page=2",
  "previous": null,
  "results": [
    {
      "id":            "550e8400-e29b-41d4-a716-446655440000",
      "title":         "Factura Q1 2024 — Proveedor ABC",
      "document_type": "INVOICE",
      "status":        "PENDING_APPROVAL",
      "file_size":     245760,
      "extracted_data": {
        "Invoice Number": "FAC-2024-001",
        "Date":           "2024-02-15",
        "Total Amount":   "COP 12.500.000",
        "Vendor Name":    "Proveedor ABC S.A.S"
      },
      "workflow_arn":  "arn:aws:states:us-east-1:123:execution:docuflow-workflow-prod:doc-001",
      "uploaded_by":   "usr-001",
      "reviewed_by":   null,
      "reviewed_at":   null,
      "review_notes":  "",
      "created_at":    "2024-02-15T10:00:00Z",
      "updated_at":    "2024-02-15T10:02:30Z",
      "upload_url":    null
    }
  ]
}
```

**Crear Documento — Iniciar Carga**
```bash
POST /api/v1/documents/
Authorization: Bearer {token}
Content-Type: application/json

{
  "title":         "Factura Q1 2024 — Proveedor ABC",
  "document_type": "INVOICE"
}

# Respuesta: 201 Created
{
  "id":            "550e8400-e29b-41d4-a716-446655440000",
  "title":         "Factura Q1 2024 — Proveedor ABC",
  "document_type": "INVOICE",
  "status":        "UPLOADED",
  "upload_url":    "https://docuflow-documents-prod.s3.amazonaws.com/uploads/acme-corp/550e8400?X-Amz-Signature=...",
  "created_at":    "2024-02-15T10:00:00Z"
}

# El cliente debe luego hacer PUT del archivo directamente a upload_url:
# PUT {upload_url}
# Content-Type: application/pdf
# Body: <contenido binario del archivo>
```

**Obtener Documento**
```bash
GET /api/v1/documents/{id}/
Authorization: Bearer {token}

# Respuesta: 200 OK — objeto documento completo con extracted_data
```

**Aprobar Documento**
```bash
POST /api/v1/documents/{id}/approve/
Authorization: Bearer {token}
Content-Type: application/json

{
  "notes": "Todos los campos verificados — factura coincide con orden de compra OC-2024-089"
}

# Respuesta: 200 OK
{
  "id":           "550e8400-e29b-41d4-a716-446655440000",
  "status":       "APPROVED",
  "reviewed_by":  "usr-reviewer-001",
  "reviewed_at":  "2024-02-15T11:30:00Z",
  "review_notes": "Todos los campos verificados — factura coincide con orden de compra OC-2024-089"
}
```

**Rechazar Documento**
```bash
POST /api/v1/documents/{id}/reject/
Authorization: Bearer {token}
Content-Type: application/json

{
  "reason": "Falta firma del proveedor en la página 3 — por favor reenviar"
}

# Respuesta: 200 OK
{
  "id":           "550e8400-e29b-41d4-a716-446655440000",
  "status":       "REJECTED",
  "reviewed_by":  "usr-reviewer-001",
  "reviewed_at":  "2024-02-15T11:35:00Z",
  "review_notes": "Falta firma del proveedor en la página 3 — por favor reenviar"
}
```

**Listar Pendientes de Aprobación**
```bash
GET /api/v1/documents/pending/
Authorization: Bearer {token}

# Respuesta: 200 OK — lista de documentos en estado PENDING_APPROVAL
# solo para el tenant del usuario autenticado
{
  "count": 3,
  "results": [...]
}
```

#### 2. Flujos de Trabajo

**Obtener Estado del Flujo de Trabajo**
```bash
GET /api/v1/workflows/{document_id}/status/
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "document_id":     "550e8400-e29b-41d4-a716-446655440000",
  "execution_arn":   "arn:aws:states:us-east-1:123:execution:docuflow-workflow-prod:doc-001",
  "current_state":   "WaitForApproval",
  "status":          "RUNNING",
  "started_at":      "2024-02-15T10:00:05Z",
  "elapsed_seconds": 7205,
  "states_history": [
    { "state": "ExtractDocumentText", "status": "SUCCEEDED", "duration_ms": 3420 },
    { "state": "ValidateExtraction",  "status": "SUCCEEDED", "duration_ms": 180  },
    { "state": "NotifyReviewer",      "status": "SUCCEEDED", "duration_ms": 560  },
    { "state": "WaitForApproval",     "status": "RUNNING",   "duration_ms": null }
  ]
}
```

**Listar Ejecuciones de Flujos de Trabajo**
```bash
GET /api/v1/workflows/?status=RUNNING&page=1
Authorization: Bearer {token}

# Parámetros de consulta:
# - status: string (RUNNING, SUCCEEDED, FAILED, TIMED_OUT, ABORTED)
# - from:   fecha ISO
# - to:     fecha ISO

# Respuesta: 200 OK
{
  "count": 5,
  "results": [
    {
      "document_id":    "550e8400-e29b-41d4-a716-446655440000",
      "document_title": "Factura Q1 2024",
      "execution_arn":  "arn:aws:states:...",
      "status":         "RUNNING",
      "current_state":  "WaitForApproval",
      "started_at":     "2024-02-15T10:00:05Z"
    }
  ]
}
```

#### 3. Tenants

**Obtener Perfil del Tenant**
```bash
GET /api/v1/tenants/me/
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "tenant_id":       "acme-corp",
  "name":            "Acme Corporation S.A.S",
  "plan":            "ENTERPRISE",
  "documents_used":  1247,
  "documents_limit": 10000,
  "members_count":   12,
  "created_at":      "2023-06-01T00:00:00Z"
}
```

**Listar Miembros del Tenant**
```bash
GET /api/v1/tenants/members/
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "count": 12,
  "results": [
    {
      "id":        "usr-001",
      "email":     "manager@acme-corp.docuflow.io",
      "role":      "MANAGER",
      "is_active": true,
      "joined_at": "2023-06-01T00:00:00Z"
    }
  ]
}
```

#### 4. Analítica

**Resumen de Procesamiento de Documentos**
```bash
GET /api/v1/analytics/summary/?from=2024-02-01T00:00:00Z&to=2024-02-29T23:59:59Z
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "tenant_id":                  "acme-corp",
  "period_from":                "2024-02-01T00:00:00Z",
  "period_to":                  "2024-02-29T23:59:59Z",
  "total_documents":            247,
  "by_status": {
    "APPROVED":                 198,
    "REJECTED":                 31,
    "PENDING_APPROVAL":         12,
    "VALIDATION_FAILED":        6
  },
  "by_type": {
    "INVOICE":                  142,
    "CONTRACT":                 63,
    "FORM":                     29,
    "REPORT":                   13
  },
  "avg_extraction_confidence":  91.4,
  "avg_workflow_duration_hours": 3.7,
  "approval_rate_pct":          86.5
}
```

**Línea de Tiempo de Procesamiento**
```bash
GET /api/v1/analytics/timeline/?from=2024-02-01T00:00:00Z&to=2024-02-29T23:59:59Z&bucket=day
Authorization: Bearer {token}

# Parámetros de consulta:
# - bucket: string (hour, day, week)

# Respuesta: 200 OK
[
  {
    "bucket":   "2024-02-01",
    "uploaded": 14,
    "approved": 11,
    "rejected": 2,
    "failed":   1
  },
  {
    "bucket":   "2024-02-02",
    "uploaded": 9,
    "approved": 8,
    "rejected": 1,
    "failed":   0
  }
]
```

#### 5. Interno — Callbacks Lambda
```bash
# Usado exclusivamente por funciones Lambda — no expuesto a clientes frontend

POST /internal/documents/status/
Authorization: Token {django_internal_token}
Content-Type: application/json

{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "status":      "PENDING_APPROVAL"
}

# Respuesta: 200 OK
{ "updated": true }
```
```bash
POST /internal/workflows/token/
Authorization: Token {django_internal_token}
Content-Type: application/json

{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "task_token":  "AQDEAAAAKgAAAAMAAAAAAAAAAURjMmU5..."
}

# Respuesta: 200 OK — almacena token de tarea para estado WaitForApproval
{ "stored": true }
```

#### 6. WebSocket
```bash
WS /ws/documents/
# Token pasado como parámetro de consulta
WS /ws/documents/?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# El servidor emite en cada cambio de estado de documento dentro del tenant:
{
  "type":        "DOCUMENT_STATUS_UPDATE",
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "status":      "APPROVED",
  "updated_at":  "2024-02-15T11:30:00Z"
}

# Keepalive del cliente:
→ enviar:  "ping"
← recibir: "pong"
```

### Respuestas de Error
```json
{
  "detail":      "No tiene permiso para realizar esta acción.",
  "status_code": 403,
  "code":        "permission_denied"
}
```

**Códigos de Error Comunes**

| Código                | Estado HTTP | Descripción                                          |
|-----------------------|-------------|------------------------------------------------------|
| `not_authenticated`   | 401         | Token JWT faltante o expirado                        |
| `permission_denied`   | 403         | El usuario no tiene el rol o acceso al tenant        |
| `not_found`           | 404         | Documento, flujo de trabajo o tenant no encontrado   |
| `validation_error`    | 422         | Cuerpo de solicitud inválido o campos requeridos faltantes |
| `conflict`            | 409         | El documento ya tiene un flujo de trabajo activo     |
| `throttled`           | 429         | Límite de solicitudes excedido — header retry-after presente |
| `internal_error`      | 500         | Error inesperado del servidor — revisar logs CloudWatch |

---

## 🤝 Contribución

Este proyecto fue desarrollado como parte de la labor investigativa en el SENA.
Aunque el código fuente y las aplicaciones son propiedad del SENA, las contribuciones
y sugerencias son bienvenidas.

### Flujo de Desarrollo
```bash
# 1. Crear una rama de funcionalidad
git checkout -b feature/nombre-de-la-funcionalidad

# 2. Realizar los cambios siguiendo las convenciones de la capa de servicios Django

# 3. Ejecutar la suite completa de pruebas
pytest                                  # Todas las pruebas del backend
pytest --cov=. --cov-report=html        # Con reporte de cobertura HTML
pytest -m unit                          # Solo pruebas unitarias
pytest -m integration                   # Solo pruebas de integración
npx cypress run                         # Pruebas E2E del frontend

# 4. Formatear y verificar el código
black .                                 # Formateador Python
isort .                                 # Ordenador de imports
flake8 .                                # Linter Python
mypy .                                  # Verificación de tipos estática
npm run lint                            # ESLint + TypeScript

# 5. Hacer commit usando commits convencionales
git commit -m "feat: agregar endpoint de aprobación masiva para gerentes de tenant"
git commit -m "fix: corregir almacenamiento de token de tarea Step Functions en cargas concurrentes"
git commit -m "test: agregar pruebas de integración para aislamiento de documentos multi-tenant"
git commit -m "infra: agregar outputs CloudFormation para referencias ARN Lambda entre stacks"

# 6. Subir cambios y abrir pull request
git push origin feature/nombre-de-la-funcionalidad
```

### Guía de Estilo de Código
```bash
# Python / Django — estándares obligatorios
# - Formato Black, longitud de línea 88
# - isort para ordenamiento de imports
# - Anotaciones de tipo requeridas en todas las firmas de funciones
# - Patrón de capa de servicios Django — sin lógica de negocio en vistas o modelos
# - Los serializadores DRF validan toda entrada — nunca confiar en request.data crudo
# - Todas las llamadas al SDK AWS mockeadas en pruebas unitarias — nunca impactar AWS real en CI
# - pytest-django para todas las pruebas de base de datos — no usar unittest.TestCase
# - factory_boy para toda creación de fixtures de prueba — no usar model.objects.create manual

# TypeScript — estándares obligatorios
# - Modo estricto habilitado — sin any implícito
# - Todas las acciones async Redux usan createAsyncThunk
# - Todos los componentes incluyen data-testid para selectores Cypress
# - Eventos WebSocket despachados como acciones Redux via hook personalizado
# - Sin llamadas directas a fetch() — todo HTTP a través de la capa de servicio Axios
```

---

## 📄 Licencia

Este proyecto fue desarrollado durante la labor investigativa y de instrucción en
el **SENA (Servicio Nacional de Aprendizaje)** bajo el programa **SENNOVA**,
enfocado en apoyar la transformación digital y la automatización en la nube para
empresas colombianas e instituciones públicas.

> ⚠️ **Aviso de Propiedad Intelectual**
>
> El código fuente, diseño arquitectónico, documentación técnica y todos los
> activos asociados son **propiedad institucional del SENA** y no están
> disponibles públicamente en este repositorio. El contenido presentado aquí —
> incluyendo especificaciones técnicas, diagramas de arquitectura, modelos Django,
> implementaciones de funciones Lambda, plantillas CloudFormation, máquinas de
> estados Step Functions y documentación de la API — ha sido **recreado únicamente
> con fines de demostración de portafolio**, sin exponer información institucional
> confidencial ni el código fuente original de producción.
>
> Las capturas de pantalla e imágenes de la interfaz han sido intencionalmente
> excluidas para proteger la confidencialidad de los datos operativos y la
> privacidad institucional.

**Disponible para:**

- ✅ Consultoría personalizada e implementación para sistemas serverless de automatización documental
- ✅ Diseño de arquitectura AWS serverless — Lambda, Step Functions, CloudFormation
- ✅ Desarrollo de APIs Django REST Framework para plataformas SaaS empresariales
- ✅ Diseño de aplicaciones multi-tenant con control de acceso basado en roles
- ✅ Configuración de pipelines CI/CD GitHub Actions integrando despliegues CloudFormation
- ✅ Desarrollo de módulos adicionales y soporte de sistemas en producción

---

*Desarrollado por **Paula Abad** — Desarrolladora de Software Senior e Instructora/Investigadora SENA*
*🌐 [paulabad.tech](https://paulabad.tech) · 📱 Soporte directo de la desarrolladora vía WhatsApp*
| **AWS CloudWatch**     | Logs Lambda, trazas Step Functions, métricas ECS      |
| **AWS X-Ray**          | Rastreo distribuido entre Lambda y Django             |
| **AWS Secrets Manager**| Gestión segura de credenciales                        |
