<template>
  <div id="canvas-wrapper" @mousedown="ClosePalette">
    <input type="file" style="display:none"
           accept="image/png, image/jpeg" >
    <canvas id="canvas" >
    </canvas>
  </div>
</template>

<script>
import { fabric } from 'fabric';
import eventHub from '../eventHub';
import store from '../store';
//canvas
let canvas;
let canvashistory = [];
let currentindex  = 0;

export default{
  data(){
    return {
      isEraserMode:false,
      isMouseDown:false,
    }
  },
  created(){
    eventHub.$on('objchanged',function(value,key){ 
      if(key != "createdAt")
        this.SocketToCanvas(value,key); 
    }.bind(this)) 
     eventHub.$on('objremoved',function(key){ 
        this.DeleteFromCanvas(key); 
    }.bind(this))    


    eventHub.$on('undobutton',function(){      
        this.UndoCanvas();
    }.bind(this))    

    eventHub.$on('stickybutton', function(){
      canvas.isDrawingMode = false;
      this.ChangeEraserMode(false);

      let user = window.prompt("表示したい文字を入力してください。", "");
      if(user == "" || user == null) {
        eventHub.$emit('pointerbutton');
        return;
      }
      
      let sticky = new fabric.Text(user,{
        left:100,
        top:280,
        fontSize:30,
        textAlign:'center',
      })
      //set id property created uniqueID
      sticky.set('id',this.CreateUniqueID());
      canvas.add(sticky);
      this.SaveCanvasHistory(sticky,'ADD',true);
      this.ArrangeDataForDB(sticky,'post');
      eventHub.$emit('pointerbutton');
    }.bind(this)),

    eventHub.$on('drawbutton',function (brush) {
      canvas.isDrawingMode = true;
      this.ChangeEraserMode(false);
      canvas.freeDrawingBrush.width = store.__getbrush();
      canvas.freeDrawingBrush.color = store.__getcolor();
    }.bind(this))

    eventHub.$on('eraserbutton',function () {
      canvas.isDrawingMode = false;
      this.ChangeEraserMode(true);
      
    }.bind(this))

    eventHub.$on('pointerbutton',function () {
      canvas.isDrawingMode = false;
      this.ChangeEraserMode(false);
    }.bind(this))

    eventHub.$on('imagebutton',function () {
      canvas.isDrawingMode = false;
      this.ChangeEraserMode(false);

      this.$el.firstChild.value = '';
      //open filespicker 
      this.$el.firstChild.click(); 

      this.$el.firstChild.onchange = function(e){
        
        let reader = new FileReader();
        reader.readAsDataURL(e.target.files[0]);
        reader.onload = function(){
        eventHub.$emit("imgupload",reader.result,e.target.files[0].name);
        }
      }
      eventHub.$emit('pointerbutton');
    }.bind(this))     

    eventHub.$on("sendedimg",function(URL){
      let scaleX = 1;
      let scaleY = 1;
      let widthscale,heightscale;
      const Regulationwidth  = 300;
      const Regulationheight = 300;

      new fabric.Image.fromURL(URL,function(img){
        if(img.getOriginalSize().width > Regulationwidth 
          && img.getOriginalSize().height > Regulationheight){
            //into appropriately sized objects
            widthscale = Regulationwidth / img.getOriginalSize().width;
            heightscale = Regulationheight / img.getOriginalSize().height;
            scaleX = widthscale < heightscale ? widthscale : heightscale;
            scaleY = widthscale < heightscale ? widthscale : heightscale;
          }
      
        img.set({
          left:200,
          top:200,
          scaleX:scaleX,
          scaleY:scaleY,
          //set id property created uniqueID
          id:this.CreateUniqueID()
        })                           
        canvas.add(img);
        this.SaveCanvasHistory(img,'ADD',true);
        this.ArrangeDataForDB(img,'post');            
      }.bind(this)); 
    }.bind(this))

    eventHub.$on('deleteall',function(){
      let arrangeobj = {_objects:[]};
      if(canvas._objects.length){
         
          canvas._objects.forEach(function(obj) {
              if(canvas._objects.length > 1){
                arrangeobj._objects.push(obj);
              }else{
                arrangeobj = obj;
              }
              canvas.fxRemove(obj);
          });   
        this.SaveCanvasHistory(this.CreateClone(arrangeobj),'REMOVE',true);
        this.ArrangeDataForDB(arrangeobj,'delete');
      }
    }.bind(this))

    eventHub.$on("changeBGcolor",function(){
      let BGcolor = store.__getBGcolor();
      canvas.backgroundColor = BGcolor;
      canvas.renderAll();
    })
  },
  mounted(){ 

    canvas = new fabric.Canvas('canvas',{
      backgroundColor: 'white',
      rotationCursor:'grab',
    }) 

    window.addEventListener('resize',this.CanvasResize,false);
    this.CanvasResize();

    fabric.Object.prototype
      .set({
        cornerStyle:'circle',
        lockScalingFlip:true,
        lockUniScaling:true,
        borderColor: 'rgb(80,210,210)',
        borderOpacityWhenMoving: 0,
        cornerSize:18,
        cornerColor:'rgb(80,210,210)',
        hasRotatingPoint:false
      })   
      
    canvas.on("mouse:move",function(e){
      if(this.isEraserMode&&this.isMouseDown&&e.target){
        canvas.remove(e.target);
        this.SaveCanvasHistory(this.CreateClone(e.target),'REMOVE',true);
        this.ArrangeDataForDB(e.target,'delete');
      }
    }.bind(this))

    canvas.on('mouse:down',function(options){      
      if(this.isEraserMode){
        this.isMouseDown = true;     
      }          
    }.bind(this))

    canvas.on('mouse:up',function(events){
      if(this.isEraserMode&&this.isMouseDown)this.isMouseDown =false;
      if(canvas.isDrawingMode){
        let drawingobj = canvas._objects[canvas._objects.length-1];
        //set id property created uniqueID
        drawingobj.set({
          id:this.CreateUniqueID()
        });        
        this.SaveCanvasHistory(drawingobj,'ADD',true);
        this.ArrangeDataForDB(drawingobj,'post');       
      }
    }.bind(this))

    canvas.on('object:modified',function(event){      
        this.SaveCanvasHistory(event.target,'MODIFY',true);    
        this.ArrangeDataForDB(event.target,'post');        
    }.bind(this))
    canvas.on({
          'selection:created': function(e){ 
            this.SaveCanvasHistory(e.target,'SELECTED',e.e);
            }.bind(this),
          'selection:updated': function(e){
            this.SaveCanvasHistory(e.target,'SELECTED',e.e)}.bind(this)});    
  },
  methods: {
    ChangeEraserMode:function(bool){
      if(bool){
        this.isEraserMode = true;
        canvas.discardActiveObject();
        canvas.renderAll();
        canvas.hoverCursor = 'allow';
        canvas.selection = false;
        canvas.forEachObject(function(object){ 
          object.selectable = false; 
        });
      }else{
        this.isEraserMode = false;
        canvas.hoverCursor = 'move';
        canvas.selection = true;
        canvas.forEachObject(function(object){ 
          object.selectable = true; 
        });        
      }
    },
    CanvasResize:function(){
      canvas.setDimensions({
        width : this.$el.offsetWidth,
        height: this.$el.offsetHeight});
        canvas.renderAll();
    },
    ClosePalette:function(){            
      if(this.$parent.$children[2]._data.activepalette)
        this.$parent.$children[2]._data.activepalette = false;

      if(this.$parent.$children[3]._data.aboutcanvasactive){
        this.$parent.$children[3]._data.aboutcanvasactive = false; 
        eventHub.$emit("pointerbutton");       
      }
    },
    CreateClone:function(objs){
      let clonearray = [];
      if(objs._objects){
        objs._objects.forEach(function(obj){
          obj.clone(function(clone){
            clone.set({id:obj.id});
            clonearray.push(clone);
          })
        })
      }else{
        objs.clone(function(clone){
          clone.set({id:objs.id});
          clonearray.push(clone);
        })
      }
      return clonearray;    
    },
    ArrangeDataForDB:function(targets,ope){
      //
      canvas.discardActiveObject();   
      let objects = [];      

      if(targets._objects && ope == 'post'){
        targets._objects.forEach(function(element){
          let wkobject = {};          
          wkobject.uniqueid = element.id;
          //deep copying
          wkobject.element  = JSON.parse(JSON.stringify(element));
          objects.push(wkobject);                             
        });
        let selection = new fabric.ActiveSelection(
        targets._objects, {canvas: canvas});        
        canvas.setActiveObject(selection); 

      }else if(ope == 'post'){
        let wkobject = {};
        wkobject.uniqueid = targets.id;
        wkobject.element  = targets;
        objects.push(wkobject);   
        canvas.setActiveObject(targets);          
            
      }else if(targets._objects && ope == 'delete'){ 
        targets._objects.forEach(function(obj){
          objects.push(obj.id);
        })

      }else{
        objects.push(targets.id);
      };      

      objects.forEach(function(obj){
        if(ope == "post"){
         eventHub.$emit("post",obj.element,obj.uniqueid); 
        }else if(ope == "delete"){
          eventHub.$emit("delete",obj); 
        }
      })
    },
    CreateUniqueID:function(myStrong){
      let strong = 1000;
      if (myStrong) strong = myStrong;
      return new Date().getTime().toString(16)  + Math.floor(strong*Math.random()).toString(16)      
    },
    SaveCanvasHistory:function(objects,ope,mouseevent){            
      if(!mouseevent&&objects._objects) return;
    //ope {'ADD'||'REMOVE'||'SELECTED'||'MODIFY'}
      let AddOrRemoveObj = {operation:'',objects :[]};
      let ModifyObj      = {operation:'',coords :[]};

      if(ope == 'ADD' || ope == 'REMOVE'){
        if(canvashistory[currentindex])canvashistory.pop();
        AddOrRemoveObj.operation = ope;
        AddOrRemoveObj.objects   = objects;
        canvashistory.push(AddOrRemoveObj);
        currentindex++;

      }else{
        if(ope == 'SELECTED'){
          
          if(canvashistory[currentindex]){
            canvashistory.pop();
            canvashistory.push(ModifyObj);
          }else{
            canvashistory.push(ModifyObj);
          }
          
          canvashistory[currentindex].operation = ope;   
          if(objects._objects){
            objects._objects.forEach(function(obj){   
 
              let testarry  = obj.ownMatrixCache.key.split('_');               
              canvashistory[currentindex].coords.push({
                id    : obj.id,
                top   : Number(testarry[0],10),
                left  : Number(testarry[1],10),
                scaleX : obj.scaleX,
                scaleY : obj.scaleY,
              })
            })            
          }else{
            canvashistory[currentindex].coords.push({
              id     : objects.id,
              left   : objects.left,
              top    : objects.top,
              scaleX : objects.scaleX, 
              scaleY : objects.scaleY,
            })
          }                  
        }else if(ope == 'MODIFY'){                    
          canvashistory[currentindex].operation = ope;      
          currentindex++;
          
          if(objects._objects){            
            canvashistory.push(ModifyObj);
            canvashistory[currentindex].operation = ope; 
            objects._objects.forEach(function(obj){ 

              canvashistory[currentindex].coords.push({
                id     : obj.id,
                left   : objects.translateX + obj.left,
                top    : objects.translateY + obj.top,
                scaleX : obj.scaleX, 
                scaleY : obj.scaleY,                
              })
            })
          }
        }        
      }  
    },
    UndoCanvas:function(){
      currentindex--;      
      if(currentindex < 0){
        currentindex++;
        return;
      };

      let svarray = {_objects:[]};
      let target = canvashistory[currentindex];  

      switch(canvashistory[currentindex].operation) {
      //have to apply a class to object (each class has 'clone function') 
        case 'ADD'    : 
          canvas._objects.forEach(function(obj){           
            if(obj.id == target.objects.id){
              canvas.remove(obj);
            }
          })    
          this.ArrangeDataForDB(target.objects,'delete');
          break;

        case 'REMOVE' : 
          target.objects.forEach(function(obj){
            obj.clone(function(clone){
              clone.set({id : obj.id});
              canvas.add(clone);
              svarray._objects.push(clone);
            })
          }) 
          if(target.objects.length == 1){
            this.ArrangeDataForDB(target.objects[0],'post');
          }else{                        
            this.ArrangeDataForDB(svarray,'post');
          }
          canvas.discardActiveObject();    

          break;

        case 'MODIFY' :        
          canvas.discardActiveObject();    

          target.coords.forEach(function(histarget){
            canvas._objects.forEach(function(obj){

              if(obj.id == histarget.id){
                obj.set({
                  left:histarget.left,
                  top:histarget.top,
                  scaleX:histarget.scaleX,
                  scaleY:histarget.scaleY
                })
                svarray._objects.push(obj);
                obj.setCoords();
              }
            })
          })     
          if(target.coords.length == 1){
            this.ArrangeDataForDB(svarray._objects[0],'post');
          }else{
            this.ArrangeDataForDB(svarray,'post');
          }
          canvas.renderAll();                  
          break;                  
      }
      canvashistory.pop();      
    },
    SocketToCanvas:function(value,key){ 
       
      
      let svindex = -1;
      canvas.discardActiveObject();  
      let castedvalue = JSON.parse(value);

      castedvalue.__proto__ = this.GetCanvasKlass(castedvalue.type);
      castedvalue.clone(function(clone){    
        clone.set({id: key});
        if(canvas._objects.length == 0){
          canvas.add(clone);
          return;
        }
        canvas._objects.forEach(function(elm,index){
            if(key == elm.id){
                canvas.insertAt(clone,index,true);
            svindex = index;
          }
        }.bind(this))  
        if(svindex == -1)canvas.add(clone);
      });
    },
    DeleteFromCanvas:function(id){
      canvas.discardActiveObject();
      canvas._objects.forEach(function(obj){
        if(obj.id == id){
          canvas.remove(obj);
        }
      })
    },
    GetCanvasKlass:function(type){
      switch(type) {
      //have to apply a class to object (each class has 'clone function') 
        case 'image': return fabric.Image.prototype; break;
        case 'text' : return fabric.Text.prototype; break;
        case 'path' : return fabric.Path.prototype; break;                  
      }
    }
  },
}
</script>

<style scoped>
  #canvas-wrapper{
    height: 100vh;
    width:100vw;
  }
</style>