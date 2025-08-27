import json
import boto3
from datetime import datetime
from decimal import Decimal

# Custom encoder to handle DynamoDB Decimal type
class DecimalEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        return super(DecimalEncoder, self).default(obj)

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('visitorcount')

def lambda_handler(event, context):
    method = event.get('httpMethod', 'GET')

    if method == 'POST':
        # Increment counter
        response = table.update_item(
            Key={'counterID': 'visitorcount'},
            UpdateExpression="SET visitCount = if_not_exists(visitCount, :start) + :inc, #ts = :timestamp",
            ExpressionAttributeValues={
                ':inc': 1,
                ':start': 0,
                ':timestamp': str(datetime.utcnow())
            },
            ExpressionAttributeNames={
                "#ts": "timestamp"
            },
            ReturnValues="UPDATED_NEW"
        )
        count = response['Attributes']['visitCount']

    else:  # GET or other
        # Read counter without incrementing
        response = table.get_item(Key={'counterID': 'visitorcount'})
        count = response.get('Item', {}).get('visitCount', 0)

    return {
        'statusCode': 200,
        'headers': {  # Required for CORS
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
            "Access-Control-Allow-Methods": "OPTIONS,POST,GET"
        },
        'body': json.dumps({'count': count}, cls=DecimalEncoder)
    }
