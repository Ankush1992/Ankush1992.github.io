---
layout: post
title: BLEH
categories: tutorial
tags: [spring-boot]
comments: true
--- 

ank


{% highlight groovy %}

@RequestMapping(value="/",method=RequestMethod.POST)
def uploadImg(@RequestParam("file") MultipartFile file , BindingResult result)
{
	def thumbnail =new Thumbnail(file :file ,
					   width: ThumbnailConstants.SCALED_WIDTH,
					   height:ThumbnailConstants.SCALED_HEIGHT)
	thumbnailValidator.validate(thumbnail,result)	
	
	try{
		if(!result.hasErrors()){
			  new ResponseEntity(thumbnailService.resizeAndSave(thumbnail)
				  	     ,HttpStatus.OK)
		}
		else{
			 new ResponseEntity(errorService.listErrors(msgSource,result),
				            ,HttpStatus.BAD_REQUEST)
		}
		
	}catch(ex){
		log.error("Unexpected error " ,ex)
		new ResponseEntity(HttpStatus.INTERNAL_SERVER_ERROR)
		
	}
}
	
{% endhighlight %}
