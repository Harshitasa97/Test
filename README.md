"# Test" 

USE [dbUSAPShipping_UAT]
GO
/****** Object:  StoredProcedure [dbo].[usap_usps_rules_overlb]    Script Date: 10/25/2024 3:47:44 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


/* Script to CReate SP usap_usps_rules */


-- Description:	To check USPS conditions for non priority order
-- DATE         By					DESCRIPTION OF REVISION             
-- =========    ===============		=======================================   
-- 16-Oct-2024  Gerry P				Initial SP creation for USPS over a pound shipping requirement 
-- =============================================
ALTER   PROCEDURE [dbo].[usap_usps_rules_overlb]
@in_whID varchar(10),
@in_channel varchar(10),
@in_priority varchar(30),
@in_addresstype varchar(30),
@in_zipcode varchar(10),
@in_ilength decimal(7,2), 
@in_iwidth decimal(7,2), 
@in_iheight	decimal(7,2), 
@in_iweight decimal(7,2),
@in_ireqlength decimal(7,2), 
@in_ireqweight decimal(7,2),
@in_ireqgirth decimal(7,2), 
@in_containertype varchar(20),
@in_vchfreightterm varchar(30),
@in_sourcecarrier varchar(40),
@in_shiptostate varchar(3),
@in_carriermode varchar(100),
@in_department nvarchar(50), 
@in_ireqwidth nvarchar(50), 
@in_ireqheight nvarchar(50), 
@IsValid int output
AS
BEGIN
	
 	SET NOCOUNT ON;

      DECLARE @volume decimal(11,2) = @in_ilength * @in_iwidth * @in_iheight     
      DECLARE @volume_limit  decimal(11,2) = 1728
	  DECLARE @defaultDivisor INT = 200;
	  DECLARE @dim_weight decimal(11,2)	  
	  SET @IsValid = 0;

  	  IF (NOT EXISTS (SELECT 1 FROM t_cp_carrier_department_configuration NOLOCK WHERE carrier = @in_vchfreightterm AND department = @in_department AND wh_id = @in_whID AND is_restricted = 1 AND	 UPPER(@in_priority) = '0'))  --For Non Priority Order
		BEGIN 	 		 		 
			IF (@in_ilength < @in_ireqlength) AND (@in_iwidth<  @in_ireqwidth) AND (@in_iheight <  @in_ireqheight) AND @volume < @volume_limit*2   
			BEGIN
						IF @volume < @volume_limit
						BEGIN
							SET @dim_weight = @in_iweight	
						END
						ELSE
						BEGIN							
							   SELECT @dim_weight = CASE 
							   WHEN ROUND(@volume/@defaultDivisor,0) >= ROUND(@in_iweight,0) THEN ROUND(@volume/@defaultDivisor,0) 
							   ELSE ROUND(@in_iweight,0)
							   END
						 END

						 IF @dim_weight  < @in_ireqweight
							SET @IsValid = 1;
			END
		END			 
		
	IF(UPPER(@in_shiptostate) in ('AS','GU','MP','PR','VI')) 	--Check for US territory Address e.g. Puerto Rico	
		SET @IsValid = 0;	

	IF @in_carriermode='ASIS' AND @in_sourcecarrier <> 'USPS'
	   SET @IsValid=0;  
 
 END 


--select * from t_stored_item where wh_id = 'GPT'